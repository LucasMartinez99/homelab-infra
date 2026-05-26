# Backups

Database backups are the single most important operational habit on this
infrastructure. Application code lives in Git. Configurations live in this
repo. User data lives only in databases — if I lose those, I can't
reconstruct them from anything.

So this is the piece I'm most careful about.

---

## The strategy in one paragraph

A cron job runs once per day on the VPS, dumps every database I care
about, uploads the dumps to a versioned S3 bucket, and exits. The bucket
has a lifecycle policy that transitions old backups to cheaper storage
and eventually deletes them. Restores are tested periodically against a
scratch container. That's the whole strategy.

---

## What gets backed up

| Database | Engine | Backup tool |
|---|---|---|
| App3 | PostgreSQL 16 | `pg_dump` |
| App1 (prod) | MySQL 8 | `mysqldump` |
| App2 (staging) | MySQL 8 | `mysqldump` |
| Mailserver | (file-based, not in scope) | `tar` |

What's **not** backed up:

- **Application code** — lives in Git, that's the source of truth
- **Docker images** — rebuilt from source on demand
- **Logs** — ephemeral by design; anything important goes to monitoring
- **Grafana dashboards** — persisted in a Docker volume; could be lost
  in a disaster, but they're all re-importable from grafana.com IDs.
  I'm aware this is a small gap.

---

## The cron job

A representative version of the script. The real one has hostnames and
bucket names that I've replaced with placeholders here:

```bash
#!/usr/bin/env bash
set -euo pipefail

# --- config (real values in /root/.backup-env, mode 600) ---
source /root/.backup-env
# Provides: S3_BUCKET, AWS_REGION, PG_PASSWORD, MYSQL_PASSWORD

STAMP=$(date -u +%Y-%m-%dT%H-%M-%SZ)
TMPDIR=$(mktemp -d)
trap 'rm -rf "$TMPDIR"' EXIT

# --- Postgres dumps ---
PGPASSWORD="$PG_PASSWORD" pg_dump \
  -h 127.0.0.1 -p 5432 -U app app3 \
  --format=custom \
  --file "$TMPDIR/app3-$STAMP.dump"

# --- MySQL dumps ---
mysqldump \
  -h 127.0.0.1 -P 3306 -u root -p"$MYSQL_PASSWORD" \
  --single-transaction --routines --triggers \
  app1 \
  > "$TMPDIR/application1-$STAMP.sql"

mysqldump \
  -h 127.0.0.1 -P 3306 -u root -p"$MYSQL_PASSWORD" \
  --single-transaction --routines --triggers \
  app2 \
  > "$TMPDIR/application2-$STAMP.sql"

# --- compress everything together ---
tar -czf "$TMPDIR/backup-$STAMP.tar.gz" -C "$TMPDIR" \
  "app3-$STAMP.dump" \
  "application1-$STAMP.sql" \
  "application2-$STAMP.sql"

# --- upload to S3 ---
aws s3 cp \
  "$TMPDIR/backup-$STAMP.tar.gz" \
  "s3://$S3_BUCKET/daily/backup-$STAMP.tar.gz" \
  --region "$AWS_REGION"

echo "Backup complete: backup-$STAMP.tar.gz"
```

The script is installed at `/usr/local/bin/backup-all` (mode 750) and
triggered by:

```cron
# /etc/cron.d/backup-databases
30 3 * * * root /usr/local/bin/backup-all >> /var/log/backup.log 2>&1
```

3:30 AM local time. After overnight low-traffic, before any morning
deploys.

---

## Why `pg_dump --format=custom` and `mysqldump --single-transaction`

Two small choices that matter:

**Postgres custom format.** Instead of plain SQL, this is a compressed
binary format. Faster to restore, lets `pg_restore` cherry-pick specific
tables or schemas if needed during a partial recovery.

**MySQL `--single-transaction`.** Without this flag, `mysqldump` takes
table-level locks during the dump, which can stall live queries against
the production database. With it, MySQL runs the entire dump inside a
single repeatable-read transaction — no locks, consistent snapshot.
Essentially mandatory for backing up a running DB.

The `--routines --triggers` flags include stored procedures and triggers,
which the default dump excludes.

---

## S3 bucket configuration

The bucket has a few features turned on that matter for backups:

| Setting | Purpose |
|---|---|
| **Versioning enabled** | An accidental delete or overwrite leaves a recoverable previous version |
| **Block public access** | Bucket is private; no public reads possible even if URLs leak |
| **Lifecycle policy** | After 30 days → Standard-IA · After 90 days → deleted |
| **SSE-S3 encryption** | Server-side encryption at rest (default since 2023) |

The lifecycle policy is what makes long-term cost predictable. Daily
backups would otherwise grow unbounded; this caps storage at ~90 days
of history while keeping the most recent 30 in the hot tier for fast
restores.

The IAM credentials used by the cron job have a narrow policy: write
access to this one bucket prefix, nothing else. If the VPS were
compromised, the attacker can write more backups but can't read the
rest of my AWS account.

---

## Restore procedure

Documented because restores are the only thing that prove backups work.

### Postgres restore

```bash
# Pull the backup
aws s3 cp s3://<bucket>/daily/backup-<stamp>.tar.gz .

# Extract
tar -xzf backup-<stamp>.tar.gz

# Restore into a scratch database
createdb app3_restore
pg_restore \
  -h 127.0.0.1 -U app -d app3_restore \
  --clean --if-exists \
  app3-<stamp>.dump
```

### MySQL restore

```bash
# Same extraction step as above, then:
mysql -u root -p app1 < application1-<stamp>.sql
```

Restores go to a `_restore` database, never overwriting production
directly. After restore, I verify row counts and a few sample queries
match expected values before deciding whether to swap.

---

## How I verify backups actually work

The single most common backup failure mode is "the cron job runs, the
file uploads, but the file is corrupt or incomplete." A backup you can't
restore from is worse than no backup — it gives false confidence.

Periodically I:

1. Pick a random backup from S3 (not the latest)
2. Run the restore procedure above into a scratch container
3. Boot a copy of the app pointed at the restored database
4. Click through a few user-facing flows

If anything fails, the backup pipeline gets a fix. So far the issues
I've caught:

- An early version of the script forgot `--single-transaction`, producing
  inconsistent dumps under load
- A period where the IAM credentials had expired and uploads were
  silently failing (no monitoring on this — added a check afterward)

---

## What I'd improve

A few honest gaps:

- **Restore verification isn't automated.** Manual checks happen
  periodically but not on a schedule. A weekly CI job that downloads
  the latest backup and restores it against a scratch container would
  close this.
- **No off-cloud copy.** All backups are in one AWS region. A region-wide
  AWS outage or account compromise would leave me with nothing. A
  secondary copy to a different provider (or even a local cold drive)
  would be appropriate disaster recovery hygiene.
- **No alerting on backup job failures.** The cron job's exit code isn't
  monitored. If it silently breaks, I find out the next time I check the
  logs. Adding a Prometheus push-gateway entry on success would let me
  alert on "no successful backup in 26 hours."

These are tracked. They'll get fixed in priority order.

---

## What this looks like in practice

A typical week of backups in S3:

```text
backup-2026-05-25T03-30-12Z.tar.gz    47.2 MB
backup-2026-05-24T03-30-08Z.tar.gz    47.0 MB
backup-2026-05-23T03-30-15Z.tar.gz    46.8 MB
backup-2026-05-22T03-30-09Z.tar.gz    46.5 MB
backup-2026-05-21T03-30-11Z.tar.gz    46.3 MB
...
```

Sizes growing gradually as the apps accumulate data. A sudden size jump
or drop would be the signal that something changed worth investigating.
