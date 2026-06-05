# SETUP.md

## Purpose

This document describes how to rebuild the homelab from scratch on a fresh Ubuntu installation.

The steps are intentionally ordered from foundational infrastructure to applications.

Do not skip ahead.

---

# Phase 0 - Fresh Server

Starting Point:

* Fresh Ubuntu Server installation
* Internet access available
* SSH access available

Verify:

```bash
lsb_release -a
hostname
free -h
df -h /
```

---

# Phase 1 - System Preparation

Update system:

```bash
sudo apt update
sudo apt upgrade -y
```

Install common utilities:

```bash
sudo apt install -y \
curl \
wget \
git \
vim \
nano \
htop \
ca-certificates \
gnupg
```

Verify:

```bash
git --version
curl --version
```

---

# Phase 2 - SSH Hardening

Verify SSH access.

Check:

```bash
ls ~/.ssh
```

Ensure key-based authentication works.

Optional:

Disable password authentication.

File:

```text
/etc/ssh/sshd_config
```

Restart:

```bash
sudo systemctl restart ssh
```

---

# Phase 3 - Install Tailscale

Install:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

Authenticate:

```bash
sudo tailscale up
```

Verify:

```bash
tailscale status
tailscale ip -4
```

Record Tailscale IP.

Example:

```text
100.125.241.3
```

---

# Phase 4 - Install Docker

Add Docker repository.

Install:

```bash
sudo apt install -y \
docker-ce \
docker-ce-cli \
containerd.io \
docker-buildx-plugin \
docker-compose-plugin
```

Verify:

```bash
docker --version
docker compose version
```

Test:

```bash
sudo docker run hello-world
```

Enable:

```bash
sudo systemctl enable docker
```

---

# Phase 5 - Create Homelab Structure

Create root folder:

```bash
mkdir -p ~/homelab
cd ~/homelab
```

Create directories:

```bash
mkdir -p \
apps \
backups \
databases \
management \
monitoring \
reverse-proxy
```

Create management directories:

```bash
mkdir -p \
management/homepage \
management/portainer \
management/dns \
management/filebrowser
```

Create monitoring directories:

```bash
mkdir -p \
monitoring/uptime-kuma
```

Initialize Git:

```bash
cd ~/homelab
git init
```

Verify:

```bash
find ~/homelab -maxdepth 3 -type d
```

---

# Phase 6 - Create Docker Networks

Create:

```bash
docker network create management_net
docker network create private_net
docker network create public_net
```

Verify:

```bash
docker network ls
```

---

# Phase 7 - Install Cockpit

Install:

```bash
sudo apt install -y cockpit
```

Enable:

```bash
sudo systemctl enable --now cockpit.socket
```

Verify:

```bash
systemctl status cockpit.socket
```

Access:

```text
https://SERVER_IP:9090
```

---

# Phase 8 - Install Traefik

Location:

```bash
~/homelab/reverse-proxy/traefik
```

Create:

```text
traefik.yml          # Static configuration
compose.yml          # Docker Compose definition
config/
└── dynamic.yml      # Dynamic routing, middlewares, Cockpit proxy
acme/                # ACME certificate storage
certs/               # Optional custom certificates
```

Deploy:

```bash
docker compose up -d
```

Verify:

```bash
docker ps
```

Dashboard:

```text
http://SERVER_IP:8080
```

---

# Phase 9 - Install Portainer

Location:

```bash
cd ~/homelab/management/portainer
mkdir data
```

Create compose file.

Deploy:

```bash
docker compose up -d
```

Verify:

```bash
docker ps
```

Access:

```text
https://SERVER_IP:9443
```

---

# Phase 10 - Install Homepage

Location:

```bash
cd ~/homelab/management/homepage
```

Create:

```text
compose.yml
site/
├── index.html
├── styles.css
├── services.json
└── assets/
```

Homepage uses nginx:alpine to serve a custom static dashboard.

Deploy:

```bash
docker compose up -d
```

Verify:

```bash
docker ps
```

---

# Phase 11 - Connect Services to Traefik

Homepage:

```text
homelab.msaquib.com
```

Portainer:

```text
portainer.homelab.msaquib.com
```

Traefik Dashboard:

```text
traefik.homelab.msaquib.com
```

Cockpit (via dynamic.yml proxy):

```text
cockpit.homelab.msaquib.com
```

Add Traefik labels to each service compose file.

Redeploy:

```bash
docker compose up -d
```

Verify routes appear in Traefik dashboard.

---

# Phase 12 - Install CoreDNS

Location:

```bash
~/homelab/management/dns
```

Create:

```text
Corefile
compose.yml
```

Purpose:

Provide wildcard DNS resolution for:

```text
homelab.msaquib.com
*.homelab.msaquib.com
```

Deploy:

```bash
docker compose up -d
```

Verify:

```bash
dig @100.125.241.3 homelab.msaquib.com
```

Expected:

```text
100.125.241.3
```

---

# Phase 13 - Configure Tailscale Split DNS

Goal:

```text
*.homelab.msaquib.com
```

should resolve through CoreDNS for all Tailscale clients.

Configure in:

```text
Tailscale Admin
→ DNS
→ Split DNS
```

Nameserver:

```text
100.125.241.3
```

Domain:

```text
homelab.msaquib.com
```

Verify from another Tailscale device.

---

# Phase 14 - Install Uptime Kuma

Location:

```bash
~/homelab/monitoring/uptime-kuma
```

Create compose file.

Route:

```text
uptime.homelab.msaquib.com
```

Configure monitoring for:

* Homepage
* Portainer
* Cockpit
* Traefik Dashboard
* Uptime Kuma
* File Browser

Configure Twilio notifications for alerts.

Add homepage card.

Deploy:

```bash
docker compose up -d
```

---

# Phase 15 - Install File Browser

Location:

```bash
~/homelab/management/filebrowser
```

Create compose file.

Route:

```text
files.homelab.msaquib.com
```

Managed path:

```text
/home/saquib/homelab
```

Add homepage card.

Deploy:

```bash
docker compose up -d
```

---

# Phase 16 - Homepage Integration

Homepage becomes central dashboard.

Add cards for all deployed services:

* Cockpit
* Portainer
* Traefik Dashboard
* Uptime Kuma
* File Browser

Future services should always be added here.

---

# Recovery Validation Checklist

After rebuild verify:

✓ Tailscale connected

✓ Docker running

✓ Cockpit accessible

✓ Portainer accessible

✓ Homepage accessible

✓ Traefik dashboard accessible

✓ CoreDNS resolving records

✓ Split DNS functioning

✓ Uptime Kuma operational

✓ File Browser accessible

✓ Twilio alerts functional

✓ All services monitored

---

# Current Architecture

```text
Tailscale
     │
     ▼
  CoreDNS
     │
     ▼
  Traefik
     │
  ┌──┼──────────────────┐
  ▼  ▼                  ▼

 management_net      private_net
                     (reserved)
 Homepage
 Portainer
 Cockpit (proxy)
 Traefik Dashboard
 Uptime Kuma
 File Browser
```
