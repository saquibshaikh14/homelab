
# Homelab Architecture & Deep-Dive Reference

This document provides a highly detailed technical breakdown of the `homeserver` infrastructure topology, operational service layers, network policies, and manual change management workflows.

---

# Vision

This homelab provides a secure, scalable, and highly maintainable single-node environment for hosting infrastructure management tools, internal monitoring, and personal applications.

### Core Architectural Priorities
* **Security:** Strict network isolation and zero open ingress ports on local home routing hardware.
* **Simplicity:** Minimal overhead utilizing single-responsibility container abstractions where practical.
* **Reproducibility:** A completely documented and version-controlled environment allowing full recovery from bare metal.
* **Separation of Concerns:** Rigid segregation between private admin zones and application boundaries.

---

# Infrastructure Specification

### Bare-Metal Server
* **Hostname:** `homeserver`
* **Operating System:** Ubuntu 26.04 LTS
* **Compute & Memory:** 7.1 GB RAM
* **Storage Array:** 232 GB SSD

### Host System Runtimes
* **Cockpit:** Installed directly onto the native host system operating layer for hardware monitoring, OS updates, and storage array diagnostics.
* **Docker Engine (v29.5.2):** The primary containerized micro-orchestration runtime handling all application lifecycle duties.

---

# Core Component Breakdown

### Tailscale
* **Purpose:** Secure remote access, wire-level encrypted overlay mesh networking, and fine-grained access control.
* **Policy Enforcement:** All administrative dashboards, core utilities, and infrastructure endpoints remain strictly dark to the public internet and bind exclusively to the private Tailscale interface.

### Traefik
* **Purpose:** High-performance reverse proxy and unified edge service router.
* **Responsibilities:**
  * Global HTTP strictly upgraded to HTTPS redirection.
  * Hostname-based multiplexing and dynamic backend routing.
  * Centralized SSL/TLS termination and automated Let's Encrypt renewal.
* **Ingress Examples:** `homelab.msaquib.com`, `portainer.homelab.msaquib.com`, `cockpit.homelab.msaquib.com`, `traefik.homelab.msaquib.com`.

### CoreDNS
* **Purpose:** Low-latency internal DNS server integrating directly with Tailscale Split-DNS meshes.
* **Responsibilities:** Resolving internal namespace wildcard subdomains locally without leaking queries or layout patterns to public upstream recursive servers.
* **Zone Strategy:**
```text
  homelab.msaquib.com
  *.homelab.msaquib.com

```

All internal pointers explicitly resolve to the homelab node via Tailscale IPs.

---

# Network Design & Isolation

The Docker engine maintains three strictly segregated bridge networks to restrict lateral node movement in the event of an application compromise.

```text
  [ Tailscale Mesh Client ]            [ Public Web Client ]
              │                                  │
              ▼                                  ▼
    [ CoreDNS Split-DNS ]               [ Cloudflare Edge Proxies ]
              │                                  │
              ▼                                  ▼
     ( Traefik Ingress )               ( Cloudflare Tunnel Daemon )
              │                                  │
              └─────────────────┬────────────────┘
                                ▼
                        [ Traefik Ingress ]
                                │
       ┌────────────────────────┼────────────────────────┐
       ▼                        ▼                        ▼
[ management_net ]        [ private_net ]          [ public_net ]
  - Traefik Dashboard       - Backend Databases      - Public Web Apps
  - Portainer UI            - Internal APIs          - Edge API Gateways
  - Homepage Dashboard      - Cache Engines          - Static Landing Pages
  - Uptime Kuma (Target)                             - Public APIs
  - File Browser (Target)

```

### Network Policies

| Network Namespace | Permitted Ingress Vectors | Connectivity Boundaries |
| --- | --- | --- |
| **`management_net`** | Accessible solely through the local proxy when originating via authenticated Tailscale nodes.| Infrastructure tooling, reverse-proxies, monitoring dashboards, and host control systems.|
| **`private_net`** | **Zero Direct Ingress.** No public or internal routing mappings allowed.| Isolated service-to-service communication only (e.g., application microservice talking to PostgreSQL backend).|
| **`public_net`** | Reserved for future controlled inbound edge traffic.| Internet-facing web utilities and production APIs (not yet provisioned).|

---

# File System Architecture

Persistent container states, configuration assets, and data blocks reside within a predictable hierarchy anchored under the primary user account:

```text
/home/saquib/homelab/
├── apps/                   # Reserved for future user-facing applications
├── backups/                # Reserved for system backups and state dumps
├── databases/              # Reserved for persistent database engine data
├── management/             # Platform administration tools
│   ├── dns/                # CoreDNS zone configuration files
│   │   ├── Corefile
│   │   └── compose.yml
│   ├── filebrowser/        # Web-based file manager
│   │   ├── compose.yml
│   │   ├── config/
│   │   └── data/
│   ├── homepage/           # Dashboard asset configs
│   │   ├── compose.yml
│   │   ├── config/
│   │   └── site/
│   └── portainer/          # Container orchestration management
│       ├── compose.yml
│       └── data/
├── monitoring/             # Infrastructure visibility, logs, metrics, alerts
│   └── uptime-kuma/
│       ├── compose.yml
│       └── data/
└── reverse-proxy/          # Ingress control and routing
    └── traefik/            # Traefik dynamic and static file configurations
        ├── acme/           # acme.json for ACME certificate storage
        ├── certs/          # Optional custom SSL certificates
        ├── compose.yml
        ├── config/
        │   └── dynamic.yml # Dynamic routing, middlewares, Cockpit proxy
        └── traefik.yml     # Static Traefik configuration

```

---

# Cryptographic Certificate Strategy

* **Certificate Authority:** Let's Encrypt.


* **Validation Layer:** ACME-automated `DNS-01` challenge via Cloudflare API integration.


* **Scope Matrix:** Wildcard layout addressing `homelab.msaquib.com` and `*.homelab.msaquib.com`.


* **Key Benefit:** Complete generation of fully trusted browser certs for internal networks without exposing open web firewall ports or publishing internal node IP records to the global internet.



---

# System Operations & Deployment Workflow

### Version Control Topology

The local Git repository acts as the absolute source of truth for configuration.

* **Tracked Assets:** Docker Compose stacks, Traefik config parameters, CoreDNS zones, custom app configurations, homepage site assets, and documentation.


* **Excluded Assets (`.gitignore`):** Runtime state volumes (`data/`, `storage/`, `db/`), plain-text `.env` credential variables, dynamic ACME `acme.json` private key elements, SSL certificates (`*.key`, `*.pem`, `*.crt`), and log files.



### Current Manual Deployment Process

The homelab utilizes a **fully manual deployment model** via CLI and web panel tooling to manage shifts carefully without unintended side effects.

```text
Local Workspace
      ↓
(git push) → GitHub Remote
                 ↓
           Homeserver Node (Manual SSH Session)
                 ↓
           [ git pull origin main ]
                 ↓
           Review Changes & Configuration Drift
                 ↓
           Portainer Web Interface → Manual Stack Update Event

```

### Fine-Grained Update Rules

Deployments adhere strictly to targeted micro-updates. Only the individual containers or components suffering from active configuration drift should experience restart sequences.

| Change Boundary Type | Operational Mitigation Strategy | Execution Interface |
| --- | --- | --- |
| **Application Config / `.env`** | Graceful process reload or micro-restart| Portainer Container Control |
| **Container Base Image Tag** | Recreate image layer map with explicit pull flags| Portainer Stack Pull/Update |
| **Compose Blueprint Modifications** | Redefine declarative stack definitions| Portainer Stack Editor |
| **Private CoreDNS Zone Record** | Send HUP signal directly to the internal DNS server daemon | `docker exec coredns kill -HUP` |
| **Proxy Dynamic Routers** | Live file system hot-reload via dynamic providers | Automatic Traefik File Watcher |

---

# Current Operational Services

### Cockpit

* **Purpose:** Baseline native host monitoring and bare-metal server system administration.

* **Capabilities:** CPU/Memory tracking, kernel updates, system logs, disk pool state execution.

* **Access Layer:** `https://cockpit.homelab.msaquib.com` (Directly installed on host, mapped through Traefik ingress proxy).

---

### Homepage

* **Purpose:** Unified single-pane-of-glass landing dashboard linking to internal stacks.

* **Capabilities:** Real-time visibility across Portainer endpoints, system health parameters, and application shortcuts.

* **Access Layer:** `https://homelab.msaquib.com`.

---

### Portainer

* **Purpose:** Centralized container orchestration interface and lifecycle control plane.

* **Capabilities:** Compose stack tracking, runtime log parsing, visual network topology mapping, volume management.

* **Access Layer:** `https://portainer.homelab.msaquib.com`.

---

### Uptime Kuma

* **Purpose:** Infrastructure health monitoring, uptime validation, SSL certificate monitoring, and service alerting.

* **Capabilities:**

  * HTTP(S) health monitoring
  * Service grouping and dashboards
  * SSL certificate expiry monitoring
  * DNS monitoring
  * Uptime reporting
  * Twilio-based alert notifications

* **Access Layer:** `https://uptime.homelab.msaquib.com`.

* **Network Placement:** `management_net`.

---

### File Browser

* **Purpose:** Secure web-based file management interface for homelab infrastructure.

* **Capabilities:**

  * Browse infrastructure directories
  * Upload/download files
  * Create and edit configuration files
  * Manage folder structures
  * Centralized access to homelab assets

* **Access Layer:** `https://files.homelab.msaquib.com`.

* **Network Placement:** `management_net`.

* **Managed Path:** `/home/saquib/homelab`.

---

### Traefik Dashboard

* **Purpose:** Live routing telemetry matrix for network edge verification.

* **Capabilities:** Inspections of active TLS configurations, middleware paths, routing maps, and service backend status.

* **Access Layer:** `https://traefik.homelab.msaquib.com`.

---

# Platform Roadmap

### Phase 1: Infrastructure & Operations (Status: Complete)

* [x] Host OS installation, Docker runtime provisioning, and Tailscale mesh join.
* [x] Setting up CoreDNS internal resolving capabilities.
* [x] Deploying Traefik proxy combined with automated Cloudflare DNS challenge certs.
* [x] Initiating core panels: Homepage, Portainer, and native Cockpit host controls.
* [x] Deploy **Uptime Kuma** for health checks, certificate monitoring, and alert notifications.
* [x] Configure **Twilio Notifications** for service outage and recovery alerts.
* [x] Deploy **File Browser** for secure web-based management of infrastructure files.
* [x] Integrate monitoring for all operational services.
* [x] Establish centralized operational visibility for core infrastructure services.

### Phase 2: Identity & Automation (Status: Planned)

* [ ] Implement **SSO Authentication** (e.g., Authentik) as a centralized Single Sign-On gateway to unify all service logins.
* [ ] Integrate Forward Authentication middlewares into Traefik to protect applications lacking native identity layers.
* [ ] Implement **DevOps Automation** for automatic deployment pipelines replacing the current manual SSH + Portainer workflow.
