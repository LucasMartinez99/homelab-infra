# Deploys

How code gets from a `git push` on my laptop to a running container on the
VPS. The goal: deploys should be **boring** — predictable, fast, and
recoverable when something breaks.

---

## The flow at a glance

```text
  git push origin main
          │
          ▼
  ┌─────────────────────┐
  │   GitHub Actions    │   Lint · test · build image
  └──────────┬──────────┘
             │
             ▼
  ┌─────────────────────┐
  │   SSH to VPS        │   Using a dedicated deploy key
  └──────────┬──────────┘
             │
             ▼
  ┌─────────────────────┐
  │   Rebuild container │   docker compose up -d --build <service>
  └──────────┬──────────┘
             │
             ▼
  ┌─────────────────────┐
  │   Health check      │   curl localhost:PORT/health
  └─────────────────────┘
```

A push to `main` of an application repo deploys that application — and
only that application — without touching anything else on the box.

---

## What I optimize for

In order of priority:

1. **Recoverability over speed.** A failed deploy should leave the box in
   a known state. I'd rather have a 4-minute deploy I can roll back than a
   30-second deploy that corrupts state.
2. **Per-service isolation.** Deploying app A should never affect app B,
   the monitoring stack, the mailserver, or anything else on the host.
3. **No deploy ceremony.** I should be able to ship a fix at 11 PM without
   thinking. If deploys require ritual, fixes get postponed.

---

## The GitHub Actions workflow

A representative workflow file lives at `.github/workflows/deploy.yml` in
each app's repo. The shape is consistent across projects:

```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
      - name: Run tests
        run: ./mvnw verify

  deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USER }}
          key: ${{ secrets.VPS_DEPLOY_KEY }}
          script: |
            cd /srv/app-name
            git pull origin main
            docker compose up -d --build app
            docker compose ps app
```

The key design choices:

- **Tests run first.** If `mvnw verify` fails, no deploy happens. This has
  saved me from shipping bad code more than once.
- **SSH with a dedicated deploy key**, not a personal one. The key has
  access only to the project's directory and Docker. If it ever leaks, the
  blast radius is bounded.
- **`--build` only the specific service.** Compose rebuilds just the changed
  image. Other containers in the stack — databases, reverse proxy sidecars —
  stay running.

---

## Secrets handling

Three layers, no secrets in the repo:

| Layer | What lives here | How it's protected |
|---|---|---|
| **GitHub Secrets** | SSH host, user, deploy key | Repo-scoped, encrypted at rest |
| **`.env` on the VPS** | DB passwords, API keys | File mode 600, outside the git working tree |
| **Docker secrets** | Sensitive runtime values (e.g. Grafana admin pwd) | Mounted as files in `/run/secrets/`, never as env vars |

The `.env` file is referenced from `docker-compose.yml` via the
`env_file:` directive. It's listed in `.gitignore` and stored at
`/srv/<app>/.env` rather than inside the repo checkout, so a `git clean -fdx`
doesn't wipe it.

---

## What I check after a deploy

The workflow includes a `docker compose ps app` at the end so the run log
shows whether the container is `running` or `restarting`. For most apps
this is sufficient.

For the URL shortener (the most user-facing service) I also do a `curl`
against the Spring Actuator health endpoint:

```bash
curl --fail --silent --show-error http://localhost:8080/actuator/health
```

If the health check fails, the workflow exits non-zero and the run shows
red in GitHub. I get a notification on my phone via GitHub's mobile app.

---

## Rollback strategy

I deliberately keep this simple:

1. `git revert <bad-commit>` on `main`
2. Push
3. The workflow runs again, redeploys the previous code

That's it. No blue/green, no canary, no traffic shifting. For a single-VPS
setup with one user-facing app, the complexity of those patterns isn't
worth their benefit. A revert-and-redeploy completes in under 4 minutes.

If the bad deploy broke things badly enough that I can't wait for a
revert, I can SSH in and `docker compose up -d --no-build <service>`
against an older image tag — but I've never actually needed to do this in
practice. The test gate catches most problems before they reach the VPS.

---

## What deploys do NOT do

A few things I intentionally keep out of the deploy path:

- **Database migrations are NOT run by the deploy.** Flyway runs at app
  startup, in the application container itself. This means a botched
  migration takes the app down but doesn't leave the deploy script in an
  ambiguous state. The trade-off: I have to be careful with destructive
  migrations.
- **No automatic dependency updates.** Renovate / Dependabot run on a
  separate PR-driven flow. Updates always go through code review (even if
  I'm the only reviewer).
- **No production deploys from feature branches.** Only `main` deploys.
  Feature branches deploy to staging if at all.

---

## Things I'd add at the next level of scale

If this setup ever grew beyond one VPS, the obvious next moves:

- **Container image registry** — currently I `--build` on the VPS from
  source. At scale you'd build once in CI, push to a registry, and pull
  on deploy.
- **Reproducible image tags** — commit SHAs as tags rather than `latest`.
  Makes rollback an explicit image swap.
- **Health checks gating traffic** — currently the reverse proxy doesn't
  drain old containers before swapping. A small window of 502s exists
  during deploy. For higher uptime I'd add a proper readiness gate.

For my current scale, the cost of those changes outweighs their benefit.
That calculus will change. The point of writing this down is so I notice
when it does.
