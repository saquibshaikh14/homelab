# Homelab Context

For AI agent understanding of the project, current progress, architecture decisions, operational workflows, and active roadmap.

---

# Overview

Personal self-hosted homelab built around:

* Tailscale
* CoreDNS
* Traefik
* Docker

Primary goal is to provide secure, private access to self-hosted services with trusted HTTPS, centralized routing, internal DNS resolution, and infrastructure monitoring.

---

# Request Flow

Administrative Services (current):

```text
User
 ↓
Tailscale
 ↓
CoreDNS
 ↓
Traefik
 ↓
Service
```

All services are private and only accessible through the Tailscale mesh network.

---

# Infrastructure Status

Phase 1: Infrastructure & Operations

```text
✅ Complete
```

Phase 2: Identity & Single Sign-On (GitHub OAuth + oauth2-proxy)

```text
✅ Deployed (Active)
```

---

# Host & System Foundation

Current Host Settings:

```text
SSH
 ├── SSH key authentication       ✅
 └── Password authentication      ❌ (Disabled)

Tailscale
 ├── Auto-start on boot           ✅
 ├── Restart on failure           ✅
 └── Connected                    ✅

Power & Sleep
 ├── Lid close action             → ignore (/etc/systemd/logind.conf)
 ├── Suspend target               → masked (disabled)
 ├── Hibernate target             → masked (disabled)
 └── Hybrid sleep target          → masked (disabled)
```

---

# Folder Structure

```text
/home/saquib/homelab/
├── apps/                   # Reserved for future applications
├── backups/                # Reserved for backups
├── databases/              # Reserved for database persistence
├── management/
│   ├── dns/
│   │   ├── Corefile
│   │   └── compose.yml
│   ├── filebrowser/
│   │   ├── compose.yml
│   │   ├── config/
│   │   └── data/
│   ├── homepage/
│   │   ├── compose.yml
│   │   ├── config/
│   │   └── site/
│   └── portainer/
│       ├── compose.yml
│       └── data/
├── monitoring/
│   └── uptime-kuma/
│       ├── compose.yml
│       └── data/
└── reverse-proxy/
    └── traefik/
        ├── acme/
        ├── certs/
        ├── compose.yml
        ├── config/
        │   └── dynamic.yml
        └── traefik.yml
```

---

# Network Architecture

```text
                         ┌────────────────────┐
                         │ Tailscale Clients  │
                         └─────────┬──────────┘
                                   │
                                   ▼
                           ┌─────────────┐
                           │   CoreDNS   │
                           └──────┬──────┘
                                  │
                                  ▼
                           ┌─────────────┐
                           │   Traefik   │
                           └──────┬──────┘
                                  │
        ┌─────────────────────────┼─────────────────────────┐
        ▼                         ▼                         ▼

 management_net            private_net               public_net
   (Active)                (Reserved)                (Reserved)

 Homepage                  Databases                 Future Apps
 Portainer                 Internal APIs
 Traefik
 Uptime Kuma
 File Browser
```

---

# DNS & TLS

Domains:

```text
homelab.msaquib.com
*.homelab.msaquib.com
```

DNS:

* CoreDNS provides internal DNS resolution.
* Tailscale Split-DNS is configured.
* Wildcard subdomains resolve correctly.

TLS:

* Let's Encrypt certificates.
* Cloudflare DNS Challenge.
* Automatic certificate renewal.
* Wildcard certificate coverage.

---

# Running Services

| Service           | URL                                   | Status |
| ----------------- | ------------------------------------- | ------ |
| Homepage          | https://homelab.msaquib.com           | ✅      |
| Portainer         | https://portainer.homelab.msaquib.com | ✅      |
| Cockpit           | https://cockpit.homelab.msaquib.com   | ✅      |
| Traefik Dashboard | https://traefik.homelab.msaquib.com   | ✅      |
| Uptime Kuma       | https://uptime.homelab.msaquib.com    | ✅      |
| File Browser      | https://files.homelab.msaquib.com     | ✅      |

Notes:

* Cockpit is installed directly on the host.
* All other services are containerized.
* Traefik is the single ingress point.
* Administrative services are protected by Tailscale.
* Individual applications currently maintain their own authentication.

---

# Monitoring & Alerting

Platform monitoring is handled through Uptime Kuma.

Monitored Services:

* Homepage
* Portainer
* Cockpit
* Traefik Dashboard
* Uptime Kuma
* File Browser

Notifications:

* Twilio notifications configured.
* Service outage alerts enabled.
* Service recovery alerts enabled.

---

# Authentication Strategy

Current Architecture:

```text
User / Tailscale
       ↓
    Traefik
       ↓ (ForwardAuth check)
 oauth2-proxy (GitHub OAuth)
       ↓ (Authenticated session)
Internal Service
```

SSO Integration:
* **Single Sign-On Gateway**: `oauth2-proxy` handles GitHub OAuth authentication for all administrative services on `management_net`.
* **Cookie Domain**: `.homelab.msaquib.com` (SSO session shared across all subdomains).
* **Traefik Middleware**: `github-auth` enforces perimeter authentication and passes identity headers (`X-Auth-Request-User`, `X-Auth-Request-Email`).
* **Service Integrations**:
  * **Homepage**: Shows authenticated GitHub user profile pill and global Sign Out button.
  * **File Browser**: Reverse proxy auto-login enabled (`auth.method: proxy`, `auth.header: X-Auth-Request-User`) directly logging into admin account.
  * **Uptime Kuma**: Internal authentication disabled (`disableAuth: true`) so perimeter SSO provides automatic direct access to the dashboard.
  * **Cockpit**: Protected at perimeter by GitHub SSO via Traefik dual-routing (`cockpit` gateway route with `github-auth` and `cockpit-internal` for PAM/WebSockets with `auth-verify`), allowing native Linux user PAM authentication.
  * **Portainer / Traefik Dashboard**: Ingress protected by GitHub SSO at the gateway.

---

# Completed

Infrastructure:

* Ubuntu Server
* Power Management & Sleep Target Masking (Always-on server)
* SSH Hardening (Key authentication enforced, password auth disabled)
* Docker
* Docker Networks
* Tailscale
* CoreDNS
* Split DNS
* Traefik
* HTTPS Routing
* Let's Encrypt
* Cloudflare DNS Challenge
* Wildcard Certificates

Management:

* Homepage
* Portainer
* Cockpit
* Uptime Kuma
* File Browser

Operations:

* Git-based configuration management
* Monitoring
* Twilio notifications

---

# Phase 2 Priorities

Priority Order:

1. SSO Authentication (unified sign-in across all services)
2. DevOps Automation (automatic deployment pipelines)

---

# Known Constraints

* Single-node deployment.
* 7.1 GB RAM.
* 232 GB SSD.
* No Kubernetes planned.
* Docker Compose is the primary deployment model.
* Deployments are currently manual.
* Portainer is used for stack management.
* Git repository is the source of truth.

---

# Important Operational Rules

* Infrastructure first.
* Private by default.
* One ingress proxy (Traefik).
* No direct public exposure of management tools.
* No blind global restarts.
* Only redeploy affected services.
* Configuration belongs in Git.
* Runtime data remains outside Git.

---

# Current Status Summary

The homelab infrastructure and operations layer is fully operational. Phase 1 is complete.

Current active stack:

```text
Homepage
Portainer
Cockpit
Traefik
CoreDNS
Uptime Kuma
File Browser
```

The next major milestones are implementing centralized SSO authentication and DevOps automation for automatic deployments.
