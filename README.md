# homelab-infra

> A case study of the self-hosted infrastructure I operate for my personal
> projects and side businesses — written up as a portfolio piece, not a
> deployable template.

This repository documents the production infrastructure I run on a single
self-managed Linux VPS: how it's architected, what it costs, what's broken
along the way, and how I keep it running. It's the operational side of being
a backend engineer who owns systems end-to-end.

All configuration examples in this repo are **sanitized** — real domains, IPs,
secrets, and hostnames have been replaced with placeholders. The architecture
and tooling, however, are exactly what's running in production right now.

---

## 🎯 Why this exists

I'm a Java / Spring Boot backend engineer. I believe writing code is half the
job — the other half is making sure it runs reliably in front of real users.
Over the past two years I've moved my personal projects from managed hosting
(cPanel) onto a self-managed VPS, and built up the operational layer myself.

This repo is the public-facing write-up of that work. It's intended for:

- 👩‍💻 **Recruiters and hiring managers** — to see how I think about
  infrastructure, beyond what's visible in application repos
- 🛠 **Other developers** — who are considering self-hosting and want a
  real-world reference
- 📝 **Future me** — documentation is the best gift you give your future self

---

## 🏗 Architecture overview

A single Ubuntu VPS runs all services. Traffic flows through Cloudflare (DNS,
TLS edge, DDoS protection) into an Nginx reverse proxy, which routes to
individual Docker Compose stacks per project.

```text
                    ┌─────────────────┐
                    │   Cloudflare    │   DNS · WAF · DDoS
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │   Nginx Proxy   │   Reverse proxy · TLS termination
                    │   (Docker)      │   (Certbot auto-renewal)
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────────────┐
        ▼                    ▼                            ▼
  ┌──────────┐         ┌──────────┐                ┌────────────┐
  │  App A   │         │  App B   │      ...       │ Mailserver │
  │ (compose)│         │ (compose)│                │  (compose) │
  └──────────┘         └──────────┘                └────────────┘

  Cross-cutting:
  • Prometheus + Grafana + cAdvisor + node_exporter  → observability
  • Cron + AWS S3                                    → automated backups
  • GitHub Actions                                   → CI/CD per app
```

A more detailed diagram lives in [`architecture/overview.png`](./architecture/overview.png).

---

## 📦 What's running

| Service | Purpose | Stack |
|---|---|---|
| **Reverse proxy** | Routes traffic, terminates TLS | Nginx + Certbot |
| **App stacks** | Several production app deployments (staging + prod) | Spring Boot / PHP / React, Docker Compose |
| **Mailserver** | Self-hosted email — no third-party email service | docker-mailserver |
| **Monitoring stack** | Metrics, dashboards, alerts | Prometheus, Grafana, cAdvisor, node_exporter |
| **Alerting** | Real-time incident notifications | Grafana Alerting → Discord webhook |
| **Backups** | Daily encrypted DB dumps → off-server storage | Cron + AWS S3 |
| **DNS / Edge** | Public-facing edge layer | Cloudflare |

---

## 🔭 Observability

The piece I'm proudest of. Before this, I had zero visibility into my own
infrastructure — if a container crashed at 3 AM, I'd find out when a user
complained. Now I have a layered observability stack that catches problems
before users do.

**Five alert rules**, deliberately kept minimal to avoid alert fatigue:

1. **Container down** — fires if any container hasn't reported metrics in 60s
2. **Host RAM > 90%** for 5 minutes
3. **Disk > 85%** full
4. **Sustained host CPU > 85%** for 10 minutes
5. **Single container hogging a full core** for 10+ minutes

Every alert has a pending period that filters out transient spikes. Each
notification is actionable — if I wouldn't act on it at 3 AM, it isn't an
alert, it's a dashboard panel.

Full configuration walkthrough in [`observability/`](./observability/).

---

## 🔐 Operational practices

What I do beyond just running containers:

- **Automated TLS** — Certbot renews certificates and reloads Nginx without
  downtime, via a cron-triggered hook
- **Daily database backups** — Cron job dumps Postgres + MySQL databases and
  uploads them to a versioned S3 bucket. Restores are tested periodically
  against a scratch container
- **Automated deploys** — GitHub Actions on push to `main`: SSH to the VPS,
  rebuild only the changed container, restart, health-check
- **Reverse proxy isolation** — every internal service stays on its compose
  network. Only Nginx is on the public network. Application containers are
  not exposed on host ports
- **Resource limits** — memory caps on critical containers so a runaway
  process can't take the host down

Details: [`operations/`](./operations/).

---

## 💸 What it costs

One of the reasons I went the self-hosting route is cost predictability:

| Item | Monthly cost |
|---|---|
| VPS (2 vCPU / 8 GB RAM / 200 GB SSD) | ~$15 |
| Domain registrations | ~$2 (amortized) |
| Cloudflare | $0 (free tier) |
| AWS S3 for backups (~5 GB) | <$1 |
| TLS certificates | $0 (Let's Encrypt) |
| **Total** | **~$18 / month** |

For comparison, the equivalent on AWS (ECS + RDS + ALB + S3 + CloudFront +
SES) would run several hundred dollars a month at the same scale. The
tradeoff: I'm on the hook for everything — there is no AWS support to call
at 3 AM. So far, I prefer it that way.

---

## 📚 Lessons learned

A few things I didn't expect:

- **Alert fatigue kills monitoring faster than no monitoring at all.** I
  started with 30 alert rules in my head, deleted 25 before turning them on.
  Each remaining alert had to pass the "would I actually act on this?" test.
- **The "no data" state is a config choice, not a bug.** Grafana defaults
  to treating empty query results as errors. You have to explicitly tell it
  that no data = healthy state.
- **Backups aren't real until you've restored from them.** Periodically I
  spin up a scratch container and restore from the latest S3 backup to
  verify the dump is usable.
- **Self-hosting email is really tough, and I love it.** SPF, DKIM, DMARC,
  reverse DNS, IP reputation — there's no abstraction that saves you from
  learning email properly. It's a rite of passage.
- **The reverse proxy is the single most important container.** If Nginx
  goes down, every service goes down. It gets the most monitoring attention
  and the strictest resource policies.

---

## 🗺 Repo structure

```text
homelab-infra/
├── README.md                 ← You are here
├── architecture/
│   └── overview.png          ← Network and service diagram
├── observability/
│   ├── prometheus.example.yml
│   ├── alerts.md             ← Full PromQL for the 5 alert rules
│   └── dashboards.md         ← Grafana dashboard IDs I use, with notes
├── operations/
│   ├── backups.md            ← S3 backup script breakdown
│   ├── tls.md                ← Certbot + Nginx setup
│   └── deploys.md            ← GitHub Actions deploy flow
└── lessons-learned.md        ← Longer-form post-mortems
```

---

## 🚧 Status

This repo is a **living document**. The infrastructure it describes evolves
continuously — when I migrate something, break something, or learn something,
I update here. Latest changes are in the commit history.

---

## 📫 About me

I'm Lucas, a backend engineer based in Paraguay 🇵🇾, focused on Java +
Spring Boot, AWS Solutions Architect Associate Certified and slowly building toward DevOps and SRE roles via the
RHCSA / CKA / Terraform Associate path.

- **GitHub**: [LucasMartinez99](https://github.com/LucasMartinez99)
- **LinkedIn**: [lucas-software-engineer](https://www.linkedin.com/in/lucas-software-engineer/)

Open to remote backend engineering opportunities.
