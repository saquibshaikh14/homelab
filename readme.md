# Homelab Platform

A secure, private, and highly reproducible personal self-hosted infrastructure platform running on a dedicated node. All services are private-by-default and accessible exclusively through a Tailscale mesh network.

## Project Mission & Strategy

The goal of this project is to shift from decentralized software setups to a structured, enterprise-grade sandbox at home. The architecture focuses on setting up a rock-solid infrastructure foundation before deploying user-facing applications.

The platform was deployed through a **two-phase implementation blueprint**:

1. **Infrastructure & Operations (Complete):** Secure network overlays, split-DNS routing, automated wildcard TLS certificates, centralized management panels, infrastructure monitoring with alerting, and web-based file management.

2. **Identity & Automation (Planned):** Introducing centralized SSO authentication to unify all service logins, and implementing DevOps automation for automatic deployment pipelines.

---

## Core Design Principles

1. **Infrastructure First:** Core networking, DNS, and proxy layers must be healthy before spinning up user applications.

2. **Private by Default:** All administrative, infrastructure, and operational tools remain locked inside the private network.

3. **One Ingress Proxy:** All traffic passes through a single reverse proxy instance (Traefik) for uniform routing, middleware application, and logging.

4. **Declarative & Version Controlled:** If a configuration or deployment pattern isn't tracked in Git, it does not exist.

5. **No Blind Global Refreshes:** Upgrades, configuration shifts, and container recreation must target only the affected micro-stack.

---

## System Specifications

### Host Infrastructure

| Resource / Variable    | Specification / Value         |
| ---------------------- | ----------------------------- |
| **Hostname**           | `homeserver`                  |
| **Operating System**   | Ubuntu 26.04 LTS              |
| **Memory Capacity**    | 7.1 GB RAM                    |
| **Storage Capacity**   | 232 GB SSD                    |
| **Container Runtime**  | Docker v29.5.2                |
| **Host Control Plane** | Cockpit (Native Installation) |

### Repository Structure

The file tree under `/home/saquib/homelab` organizes stacks logically by operational boundaries:

```text
/home/saquib/homelab/
├── apps/                   # Reserved for future applications
├── backups/                # Reserved for system backups
├── databases/              # Reserved for database persistence
├── management/             # Platform administration tools
│   ├── dns/                # CoreDNS zone configuration
│   │   ├── Corefile
│   │   └── compose.yml
│   ├── filebrowser/        # Web-based file manager
│   │   ├── compose.yml
│   │   ├── config/
│   │   └── data/
│   ├── homepage/           # Dashboard landing page
│   │   ├── compose.yml
│   │   ├── config/
│   │   └── site/
│   └── portainer/          # Container orchestration management
│       ├── compose.yml
│       └── data/
├── monitoring/             # Infrastructure visibility and alerting
│   └── uptime-kuma/        # Service status tracker
│       ├── compose.yml
│       └── data/
└── reverse-proxy/          # Ingress control and routing
    └── traefik/            # Traefik reverse proxy
        ├── acme/           # ACME certificate storage
        ├── certs/          # Optional custom certificates
        ├── compose.yml
        ├── config/
        │   └── dynamic.yml # Dynamic routing, middlewares, Cockpit proxy
        └── traefik.yml     # Static Traefik configuration
```

---

## Network Architecture & Service Routing

The environment isolates operational risks by splitting the Docker engine into three strictly segregated bridge networks:

```text
  [ Private Client ]                 [ Public Client ]
          │                                  │
   ( Tailscale VPN )                 ( Cloudflare Edge )
          │                                  │
          ▼                                  ▼
   [ CoreDNS Zone ]                  [ Cloudflare Tunnel ]
          │                                  │
          └───────────────┬──────────────────┘
                          ▼
                  [ Traefik Ingress ]
                          │
     ┌────────────────────┼────────────────────┐
     ▼                    ▼                    ▼

[ management_net ]  [ private_net ]     [ public_net ]
 - Traefik Dash      - Databases         - Public Web Apps
 - Portainer         - Internal APIs     - Edge APIs
 - Homepage          - Cache Layers
 - Cockpit (Proxy)
 - Uptime Kuma
 - File Browser
```

### Network Policies

* **`management_net`**: Dedicated to administrative panels and operational tooling. Accessible exclusively through Traefik when originating from authenticated Tailscale clients.

* **`private_net`**: Completely dark network. Contains no ingress routing definitions and exists solely for service-to-service communication.

* **`public_net`**: Reserved for future controlled internet-facing services.

---

## Security & Traffic Ingress Model

### Ingress Flow

* **All Services:** `User` → `Tailscale` → `CoreDNS` → `Traefik` → `Target Service`

### Certificate & Cryptographic Strategy

* **CA Provider:** Let's Encrypt

* **Validation Topology:** ACME Automated DNS-01 Challenge via Cloudflare DNS APIs

* **Scope:** Single wildcard certificate covering:

```text
homelab.msaquib.com
*.homelab.msaquib.com
```

* **Execution Benefit:** Browser-trusted HTTPS for internal services without exposing home infrastructure to the public internet.

---

## Live Service Matrix

All hostnames resolve correctly only when connected to the Tailscale mesh network.

| Service          | Subdomain URL                           | Security Layer      | Networking Backend | Purpose                                |
| ---------------- | --------------------------------------- | ------------------- | ------------------ | -------------------------------------- |
| **Homepage**     | `https://homelab.msaquib.com`           | Private (Tailscale) | `management_net`   | Primary single-pane landing dashboard  |
| **Portainer**    | `https://portainer.homelab.msaquib.com` | Private (Tailscale) | `management_net`   | Container lifecycle management         |
| **Cockpit**      | `https://cockpit.homelab.msaquib.com`   | Private (Tailscale) | Host Network       | Native OS administration               |
| **Traefik**      | `https://traefik.homelab.msaquib.com`   | Private (Tailscale) | `management_net`   | Reverse proxy telemetry and routing    |
| **Uptime Kuma**  | `https://uptime.homelab.msaquib.com`    | Private (Tailscale) | `management_net`   | Infrastructure monitoring and alerting |
| **File Browser** | `https://files.homelab.msaquib.com`     | Private (Tailscale) | `management_net`   | Web-based file management              |

---

## Deployment & Operations

The configuration repository serves as the absolute single source of truth for the environment. Runtime state and persistent secrets are explicitly kept out of version control.

### Version Control Matrix

**Tracked in Git**

* Docker Compose definitions
* Traefik static and dynamic routing rules
* CoreDNS zones
* Homepage site assets and configuration
* Documentation (ARCHITECTURE.md, CONTEXT.md, SETUP.md, readme.md)

**Git Ignored**

* Runtime volumes (`data/`, `storage/`, `db/`)
* `.env` files containing secrets
* Let's Encrypt ACME storage (`acme.json`)
* SSL certificates (`*.key`, `*.pem`, `*.crt`)
* Log files

### Current Manual Deployment Workflow

```text
[ Local Workspace ] ──► ( Git Push ) ──► [ GitHub Remote ]
                                                │
                                                ▼
                                         ( Manual SSH Pull )
                                                │
                                                ▼
[ Portainer UI ] ◄── ( Manual Redeploy ) ── [ Home Server Disk ]
```

1. Modify configuration locally.
2. Commit and push to GitHub.
3. SSH into `homeserver`.
4. Pull changes.
5. Redeploy only affected services through Portainer.

| Change Event Scope               | Mitigation Action             | Target Tooling                  |
| -------------------------------- | ----------------------------- | ------------------------------- |
| Application Environment Variable | Microservice Process Restart  | Portainer Container Actions     |
| New Container Image Tag          | Recreate + Pull Latest Layers | Portainer Stack Update          |
| Compose Infrastructure Shift     | Declarative Stack Overwrite   | Portainer Stack Editor          |
| Local Private Record Entry       | CoreDNS Zone Reload           | `docker exec coredns kill -HUP` |
| Ingress Rule Modifications       | Traefik Dynamic Watcher       | Automatic Hot Reload            |

---

## Platform Roadmap

### Phase 1: Infrastructure & Operations (Status: Complete)

* [x] Ubuntu Server Provisioning
* [x] Docker Runtime Installation
* [x] Tailscale Mesh Network
* [x] CoreDNS Internal Resolution
* [x] Traefik Reverse Proxy
* [x] Wildcard HTTPS Certificates
* [x] Homepage Dashboard
* [x] Portainer Container Management
* [x] Cockpit Host Administration
* [x] Uptime Kuma Monitoring
* [x] Twilio Alert Notifications
* [x] File Browser Deployment
* [x] Centralized Operational Visibility

### Phase 2: Identity & Automation (Status: Planned)

* [ ] SSO Authentication — Centralized Single Sign-On to unify all service logins
* [ ] Traefik Forward Authentication — Middleware integration for SSO-protected services
* [ ] DevOps Automation — Automatic deployment pipelines replacing manual SSH + Portainer workflow
