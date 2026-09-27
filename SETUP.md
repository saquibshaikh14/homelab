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

Verify SSH access:

```bash
ls -la ~/.ssh
```

Ensure key-based authentication works before disabling password authentication.

Disable password authentication:

File:

```text
/etc/ssh/sshd_config
```

Set:

```text
PasswordAuthentication no
PubkeyAuthentication yes
```

Restart and verify:

```bash
sudo systemctl restart ssh
sudo sshd -T | grep -i passwordauthentication
```

---

# Phase 3 - Power Management & Sleep Prevention

Configure the system to prevent suspension or sleep (crucial when using laptop or headless hardware as an always-on server):

1. Ignore lid switches in logind:

File:

```text
/etc/systemd/logind.conf
```

Configuration:

```ini
[Login]
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
```

Apply logind changes:

```bash
sudo systemctl restart systemd-logind
```

2. Mask all systemd sleep and suspend targets:

```bash
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

Verify masked status:

```bash
systemctl status sleep.target suspend.target hibernate.target hybrid-sleep.target
```

---

# Phase 4 - Install Tailscale

Install:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

Enable auto-start on boot:

```bash
sudo systemctl enable --now tailscaled
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
100.105.235.112
```

---

# Phase 5 - Install Docker

Check and remove any conflicting/legacy packages:

```bash
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove -y $pkg 2>/dev/null; done
```

Set up Docker's official apt repository:

```bash
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
```

Install Docker CE, CLI, and plugins:

```bash
sudo apt install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

Configure non-root user access (optional):

```bash
sudo usermod -aG docker $USER
```

Verify installation:

```bash
docker --version
docker compose version
```

Test:

```bash
sudo docker run hello-world
```

Enable auto-start on boot:

```bash
sudo systemctl enable docker
sudo systemctl enable containerd
```

---

# Phase 6 - Create Homelab Structure

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

# Phase 7 - Create Docker Networks

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

# Phase 8 - Install Cockpit

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

# Phase 9 - Install Traefik

Location:

```bash
cd ~/homelab/reverse-proxy/traefik
```

Structure & prerequisites:

```text
traefik.yml          # Static configuration
compose.yml          # Docker Compose definition
config/
└── dynamic.yml      # Dynamic routing, middlewares, Cockpit proxy
acme/                # ACME certificate storage (requires chmod 600 acme.json)
certs/               # Custom certificates storage
.env                 # Cloudflare DNS API token (git-ignored)
```

Initialize storage and strict permissions:

```bash
mkdir -p acme certs
touch acme/acme.json
chmod 600 acme/acme.json
```

Configure Cloudflare DNS API token:

Create `.env`:

```bash
echo "CF_DNS_API_TOKEN=your_cloudflare_api_token" > .env
chmod 600 .env
```

Deploy:

```bash
docker compose up -d
```

Verify:

```bash
docker ps
docker compose logs -f
```

Verify Let's Encrypt certificate acquisition:

```bash
ls -la acme/acme.json
```

Global HTTP → HTTPS redirection is enabled on the `web` entryPoint (`:80` → `:443`). Any unencrypted request automatically redirects to secure HTTPS.

Dashboard:

```text
http://SERVER_IP:8080
```

---

# Phase 10 - Install Portainer

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

Access (via Traefik HTTPS):

```text
https://portainer.homelab.msaquib.com
```

---

# Phase 11 - Install Homepage

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

# Phase 12 - Connect Services to Traefik

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

# Phase 13 - Install CoreDNS

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
dig @100.105.235.112 homelab.msaquib.com
```

Expected:

```text
100.105.235.112
```

---

# Phase 14 - Configure Tailscale Split DNS

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
100.105.235.112
```

Domain:

```text
homelab.msaquib.com
```

Verify from another Tailscale device:

```bash
dig @100.100.100.100 homelab.msaquib.com
```

> **Important Rebuild Note**: Whenever the homelab server is re-installed, rebuilt, or changes Tailscale IPs, you **must update the Split DNS Nameserver IP** in the Tailscale Admin Console (`Tailscale Admin → DNS → Split DNS → homelab.msaquib.com`). Otherwise, Tailscale MagicDNS will attempt to query the decommissioned IP and client browsers will encounter DNS timeouts (`DNS_PROBE_FINISHED_NXDOMAIN`).

---

# Phase 15 - Install Uptime Kuma

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
cd ~/homelab/monitoring/uptime-kuma
mkdir -p data
docker compose up -d
```

---

# Phase 16 - Install File Browser

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
cd ~/homelab/management/filebrowser
mkdir -p data
docker compose up -d
```

---

# Phase 17 - Homepage Integration

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
