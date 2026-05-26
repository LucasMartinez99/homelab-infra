# Architecture overview

The detailed view of how traffic, data, and operational concerns flow
across the VPS. The high-level summary lives in the
[main README](../README.md); this document is for readers who want the
specifics.

---

## The big picture

```text
                                ┌──────────────────────────┐
                                │       END USERS          │
                                └────────────┬─────────────┘
                                             │ HTTPS
                                             ▼
                                ┌──────────────────────────┐
                                │       CLOUDFLARE         │
                                │  · DNS                   │
                                │  · TLS at edge           │
                                │  · WAF / DDoS protection │
                                │  · Caching for static    │
                                └────────────┬─────────────┘
                                             │ HTTPS (origin pull)
                                             ▼
┌────────────────────────────────────────────────────────────────────┐
│  THE VPS (Ubuntu LTS · single host)                                │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  NGINX REVERSE PROXY (Docker)                                │  │
│  │  · Routes by virtual host                                    │  │
│  │  · Terminates origin TLS (Certbot / Let's Encrypt)           │  │
│  │  · Only container exposing host ports (80, 443)              │  │
│  └────────┬──────────────────┬───────────────┬──────────────┬───┘  │
│           │                  │               │              │      │
│           ▼                  ▼               ▼              ▼      │
│   ┌─────────────┐    ┌─────────────┐  ┌────────────┐  ┌─────────┐  │
│   │  App stack  │    │  App stack  │  │ Monitoring │  │  Mail   │  │
│   │     A       │    │     B       │  │   stack    │  │ server  │  │
│   │             │    │             │  │            │  │         │  │
│   │ ┌─────────┐ │    │ ┌─────────┐ │  │ Prometheus │  │  SMTP   │  │
│   │ │ nginx   │ │    │ │ nginx   │ │  │  Grafana   │  │  IMAP   │  │
│   │ ├─────────┤ │    │ ├─────────┤ │  │  cAdvisor  │  │  POP3   │  │
│   │ │ PHP-FPM │ │    │ │ Java app│ │  │ node_exp.  │  │ Submis. │  │
│   │ ├─────────┤ │    │ ├─────────┤ │  └────────────┘  └─────────┘  │
│   │ │ MySQL   │ │    │ │Postgres │ │                                │
│   │ └─────────┘ │    │ └─────────┘ │                                │
│   └─────────────┘    └─────────────┘                                │
│                                                                    │
│   Per-stack Docker network — internal services not host-exposed    │
│                                                                    │
└──────────────────────┬─────────────────────────────────────────────┘
                       │
                       │ daily cron · aws s3 cp
                       ▼
            ┌──────────────────────┐
            │     AWS S3 BUCKET    │
            │  · Versioning        │
            │  · 90-day lifecycle  │
            │  · Private, SSE-S3   │
            └──────────────────────┘

            ┌──────────────────────┐
            │      DISCORD         │  ◄── alerts via webhook
            └──────────────────────┘
```

---

## Traffic flow, in detail

When a user requests one of the apps:

1. **DNS resolves through Cloudflare.** Cloudflare's nameservers return
   one of their proxy IPs, not the VPS's real IP. The VPS's actual
   address isn't in public DNS at all.
2. **TLS terminates at Cloudflare's edge.** Cloudflare handles the
   user-facing certificate. This is what gives me a green padlock without
   any cert configuration on the VPS for the user-facing layer.
3. **Cloudflare makes an origin request to the VPS over HTTPS.** This is
   a second TLS connection, using a cert managed by Certbot on the VPS.
   This means even traffic between Cloudflare and my server is encrypted.
4. **The host's port 443 is owned by the Nginx reverse proxy container.**
   No other container binds to host ports.
5. **Nginx routes by Host header.** Each app has a server block matching
   its hostname; matched traffic gets proxied to the app's internal
   container.
6. **The app responds**, the response travels back the same path in
   reverse.

The reverse proxy is the **only** container with public ports. Application
containers, databases, and the monitoring stack all communicate over
internal Docker networks. None of them is reachable from the public
internet directly.

---

## Network isolation

Each project lives on its own Docker Compose network. The reverse proxy
container is on multiple networks:

```text
nginx-proxy ─┬─ default (public) ─┐
             ├─ app-a-net ────────┤
             ├─ app-b-net ────────┤── only path between
             ├─ monitoring-net ───┤   public and internals
             └─ mail-net ────────-┘
```

This means:

- App A cannot reach App B's database directly
- The monitoring stack can reach all containers (it needs to for scraping)
  but nothing can reach into monitoring's internal services
- A compromised app container can't pivot to other apps' data

---

## Where each component runs

| Component | Type | Where state lives | Restart policy |
|---|---|---|---|
| Nginx reverse proxy | Docker container | Stateless | `unless-stopped` |
| Application code | Docker container | Stateless | `unless-stopped` |
| PostgreSQL | Docker container | Named volume | `unless-stopped` |
| MySQL | Docker container | Named volume | `unless-stopped` |
| Prometheus | Docker container | Named volume (TSDB) | `unless-stopped` |
| Grafana | Docker container | Named volume (config + dashboards) | `unless-stopped` |
| cAdvisor | Docker container | Stateless | `unless-stopped` |
| node_exporter | Docker container | Stateless (reads host) | `unless-stopped` |
| Mailserver | Docker container | Named volumes (mail + state) | `unless-stopped` |
| Certbot | Cron job on host | `/etc/letsencrypt` on host | n/a |
| Backup script | Cron job on host | n/a | n/a |

All named volumes are under `/var/lib/docker/volumes/`. The full path of
each is referenced by name in the project's compose file.

`restart: unless-stopped` means: restart on crash, restart on reboot,
but don't restart if I explicitly stopped the container with
`docker stop`. This is what I want — automatic recovery from failures,
manual stops respected during maintenance.

---

## Why this shape

A few design decisions worth calling out:

**Why a single VPS instead of multiple smaller ones.** Cost mostly. A
single 2 vCPU / 8 GB VPS at ~$15/month outperforms three smaller VPSes
at the same price and gives me one host to monitor rather than three.
The downside — single point of failure — is real and accepted at this
scale. I'll split when I outgrow the box, not before.

**Why Docker Compose instead of Kubernetes.** Kubernetes' value is in
managing many nodes, doing rolling deploys across them, handling node
failure transparently, etc. None of that applies to a single VPS.
Docker Compose gives me 90% of the value (declarative services,
networking, volumes) at 10% of the operational overhead. I'll learn
Kubernetes through CKA prep — but I'll learn it on a real multi-node
cluster, not by force-fitting it here.

**Why Cloudflare in front instead of direct.** Three reasons:
1. Hides the VPS's real IP — random scanners hit Cloudflare, not me
2. DDoS protection for free at the volume I see
3. Static caching reduces origin load for assets

**Why self-host mail instead of using a service like SendGrid or SES.**
Honest answer: I started it as an exercise to learn email properly, and
once it was working there was no reason to migrate. It runs on the same
box at near-zero marginal cost. The hidden cost is IP reputation
management — see the `lessons-learned.md` for that story.

---

## Things this architecture deliberately does NOT have

For completeness, a few things that would be in a more mature setup:

- **Load balancer.** One VPS, no need.
- **Read replicas.** Read load is well within primary DB capacity.
- **Object storage for user uploads.** Apps don't currently accept user
  file uploads at meaningful volume. When they do, S3 is the answer.
- **Centralized logging.** Logs live where they're written. For larger
  scale, Loki + Promtail would slot in next to the existing Grafana.
- **Service mesh.** Wildly out of scope for this size.
- **Auto-scaling.** Vertical scale via VPS upgrade; no horizontal scaling
  in place. Will become relevant if I split apps onto separate hosts.

Each of these is a deliberate non-decision, not an oversight.
