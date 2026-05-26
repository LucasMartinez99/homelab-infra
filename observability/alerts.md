# Alert rules

The 5 alert rules running on my Prometheus + Grafana stack. Each one had to
pass the **"would I actually act on this at 3 AM?"** test before being
turned on. Alert fatigue is the silent killer of monitoring systems, so
fewer, sharper rules beat dense coverage every time.

All alerts evaluate every 1 minute and notify via a Discord webhook.

---

## Why these five, and not thirty

When I first wired up alerting, I listed every possible alert I could
imagine — disk I/O saturation, network packet loss, individual process
counts, container restart loops, swap usage, file descriptor exhaustion,
and so on. About thirty rules total.

Then I asked one question of each: **"If this alert woke me up at 3 AM,
what would I actually do?"**

If the answer was "wait until morning," it wasn't an alert — it was a
dashboard panel. If the answer was "investigate after the next deploy,"
same thing. Only five made the cut. The rest live as graphs I check
during routine reviews.

This pruning has a real cost — there are categories of problems I won't
catch instantly. I've accepted that tradeoff. The alternative — getting
notified for things I'd ignore — turns alerting into noise, and noise
turns into "mute the channel," which turns into missing the real fire.

---

## 1. Container down

**Fires when** any container hasn't been seen by cAdvisor for over 60 seconds.

```promql
(time() - container_last_seen{name!=""}) > bool 60
```

**Pending period**: 2 minutes
**Severity**: critical

Why `> bool 60` instead of just `> 60`: the `bool` modifier turns the
comparison into a 0-or-1 result per container, so the query always returns
data. Without it, Grafana sees "no data" when everything is healthy and
treats that as an error state.

**False positive cases I've seen**: container restarts during a deploy.
The 2-minute pending period filters those out — by the time we'd fire,
the new container is already reporting metrics again.

---

## 2. Host RAM high

**Fires when** host memory usage exceeds 90%.

```promql
((1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100) > bool 90
```

**Pending period**: 5 minutes
**Severity**: warning

Note the use of `MemAvailable` rather than `MemFree`. On Linux,
`MemAvailable` is the kernel's own estimate of memory available for new
allocations — it accounts for reclaimable cache and is what `free -h`
shows as "available." Alerting on `MemFree` would fire constantly because
healthy Linux systems use most "free" memory as cache by design.

---

## 3. Disk almost full

**Fires when** any real disk filesystem is over 85% full.

```promql
(1 - (
  node_filesystem_avail_bytes{fstype!~"tmpfs|fuse.lxcfs|overlay|squashfs"}
  /
  node_filesystem_size_bytes{fstype!~"tmpfs|fuse.lxcfs|overlay|squashfs"}
)) * 100 > bool 85
```

**Pending period**: 10 minutes
**Severity**: critical

The `fstype!~` filter excludes:
- `tmpfs` — RAM-backed temporary filesystems, fill up instantly and that's fine
- `overlay` — Docker's overlay filesystem, reflects image layers, not real usage
- `squashfs` — Snap package mounts, always "100% full" by design
- `fuse.lxcfs` — container view of host stats, redundant with real metrics

Without these exclusions, the alert fires constantly for non-issues. This
is a config detail I learned the hard way after my first 24 hours of
notifications.

The 85% threshold gives me roughly a day of warning at typical growth
rates — enough time to either expand the disk or clean up old data.

---

## 4. Host CPU sustained high

**Fires when** average CPU utilization across all cores exceeds 85% for
10 consecutive minutes.

```promql
(100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)) > bool 85
```

**Pending period**: 10 minutes
**Severity**: warning

Why such a long pending period: **CPU is naturally spiky**. A backup
running, a heavy database query, a JVM doing GC, a deploy compiling —
these can briefly peg CPU to 100% and that's completely normal.

The signature of a *real* problem is sustained high CPU — runaway loops,
mining malware, a DDoS, a thread pool deadlock. Those don't resolve in
30 seconds. The 10-minute pending period selects for problems worth
acting on while ignoring the noise of normal operation.

This was tuned after my first weekend with the alert at a 2-minute
pending period — I got six false alarms for completely healthy spikes.

---

## 5. Container hogging a full core

**Fires when** a single container has used over 90% of one CPU core
sustained for 10 minutes.

```promql
sum by (name) (rate(container_cpu_usage_seconds_total{name!=""}[5m])) * 100 > bool 90
```

**Pending period**: 10 minutes
**Severity**: warning

`container_cpu_usage_seconds_total` is a counter of CPU seconds consumed.
`rate(...[5m])` gives the per-second rate over the last 5 minutes.
Multiplied by 100, you get "% of one full core in use." So `> 90` means
a container is pegging a full core continuously.

On a multi-core host this isn't necessarily an emergency — the host has
other cores free — but it's almost always interesting. Either the
container is doing legitimate heavy work (in which case I should know
about it), or it has a runaway thread (in which case I definitely need
to know).

For higher thresholds on a multi-core box, swap the `> 90` for
`> 180` (almost two full cores) or higher depending on host size.

---

## Grafana configuration notes

A few things I had to configure beyond just writing the PromQL:

**No-data handling.** Grafana's default behavior when a query returns
no data is to put the alert into a `NoData` state, which is treated as
an error. For alerts where "no data = healthy" (like the container-down
alert before using `> bool`), this needs to be explicitly configured:

```
Alert state if no data: Normal
Alert state if execution error: Normal
```

Without this, my alerts list looked like a dashboard of errors even
though nothing was actually wrong.

**Evaluation group.** All five rules live in a single evaluation group
called `1m`, which means Prometheus evaluates them in series, once per
minute. For five rules this is fine — even on a small VPS, evaluation
takes well under a second.

**Folder organization.** Everything goes in a `System Alerts` folder.
When I add application-specific alerts later (e.g. `App: error rate >
5%`), they'll go in their own per-app folders so I can route them to
different Discord channels via notification policies.

---

## What's deliberately NOT alerted on

Things I monitor in dashboards but don't alert on:

- **Network I/O** — too variable to set a useful threshold
- **Disk I/O latency** — would need per-disk baselines I haven't established
- **Container restart count** — captured by "container down" indirectly
- **Cert expiry** — Certbot handles renewal automatically; I'd rather monitor the renewal job's success than the cert expiry itself
- **HTTP error rate** — application-level, belongs with the app, not infra
- **DB query slowness** — same: application-level concern

The pattern: I alert on host-level liveness and resource exhaustion. I
graph everything else.

---

## Future additions

Rules I'll likely add as the setup grows:

- **Certbot renewal failure** — if the renewal cron job exits non-zero
- **Backup job failure** — if the S3 upload cron job exits non-zero or
  uploads a suspiciously small file
- **Mail queue backup** — if Postfix's outbound queue grows beyond a
  threshold (a common indicator of email reputation problems)
