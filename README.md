# DataNest: Enterprise-Grade Home Lab & Edge Infrastructure

[![Proxmox VE](https://img.shields.io/badge/Hypervisor-Proxmox%20VE%208.x-E57008?logo=proxmox&logoColor=white)](https://www.proxmox.com)
[![Docker](https://img.shields.io/badge/Container-Docker%20%26%20Compose-2496ED?logo=docker&logoColor=white)](https://www.docker.com)
[![Cloudflare Zero Trust](https://img.shields.io/badge/Security-Cloudflare%20Zero%20Trust-F38020?logo=cloudflare&logoColor=white)](https://www.cloudflare.com)
[![Tailscale ZTNA](https://img.shields.io/badge/VPN-Tailscale%20WireGuard-242424?logo=tailscale&logoColor=white)](https://tailscale.com)
[![Ghost CMS](https://img.shields.io/badge/CMS-Ghost%205.x-15171A?logo=ghost&logoColor=white)](https://ghost.org)
[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

A production-grade, highly resilient home server and edge infrastructure setup built on a **Lenovo ThinkCentre M910q** running **Proxmox VE**. 

This repository documents the complete end-to-end architecture, configuration files, network routing strategies, security hardening protocols, and troubleshooting playbooks for running self-hosted enterprise workloads behind strict **Carrier-Grade NAT (CGNAT)** limitations.

---

## Architecture Overview

DataNest bridges local high-speed LAN services with secure public web endpoints without exposing open inbound ports (`80`/`443`) on the perimeter cellular gateway.

### High-Level System Architecture
![DataNest Architecture Diagram](docs/images/datanest-architecture-no-ip.png)

> *Note: For detailed IP addressing schemes and port mappings, refer to [docs/images/datanest-architecture-with-ip.png](docs/images/datanest-architecture-with-ip.png).*

---

## Key Features & Technical Highlights

* **Proxmox VE Bare-Metal Virtualization**: Isolated execution environments using LXC containers for core infrastructure services and a dedicated Ubuntu 24.04 LTS Virtual Machine for Docker workloads.
* **Live Zero-Downtime LVM Storage Expansion**: Dynamically expanded root filesystem (`ubuntu--vg-ubuntu--lv`) under active production load without system reboot.
* **Split-Horizon DNS & Local Wire-Speed Routing**: **Pi-hole v6** parses targeted domain overrides (`/etc/dnsmasq.d/`) to resolve `*.datanest.my.id` requests directly to the internal Reverse Proxy at LAN speed.
* **Wildcard SSL via DNS-01 API Challenge**: Automated Let's Encrypt certificates issued via Nginx Proxy Manager using Cloudflare DNS API tokens—eliminating the need for HTTP-01 inbound validation.
* **Zero Trust Ingress & CGNAT Bypass**: 
  * **Public Access**: Outbound **Cloudflare Zero Trust Tunnel (`cloudflared`)** exposes the public Ghost CMS site (`datanest.my.id`).
  * **Admin Hardening**: Sensitive routes (`/ghost*`) protected behind **Cloudflare Access Email OTP**.
  * **Remote Access (ZTNA)**: **Tailscale Subnet Router** advertises the LAN subnet over encrypted WireGuard mesh networking.
* **Enterprise SaaS Alternatives**:
  * **Vaultwarden**: Centralized credential management compatible with Bitwarden clients.
  * **Stirling PDF**: Local OCR and sensitive document processing.
  * **Immich DAM**: AI-powered photo/video asset management with ML thread limits and custom storage formatting (`{{y}}/{{MM}}/{{filename}}`).
  * **Nextcloud Cluster**: High-availability file synchronization backed by PostgreSQL and Redis memory caching.

---

## Hardware & Subsystem Specifications

| Component | Specification | Description |
| :--- | :--- | :--- |
| **Compute Host** | Lenovo ThinkCentre M910q | Intel Core i3 Gen 7, 16GB RAM, Gigabit NIC |
| **Hypervisor OS** | Proxmox VE 8.x | 256GB NVMe SSD (System & Containers) |
| **Primary Storage** | 1TB External WD SSD | Mounted at `/mnt/wd-ssd` (Samba SMB & Docker Volumes) |
| **WAN Gateway** | TP-Link MR100 (Telkomsel 4G LTE) | CGNAT Network with forced IPv4-only APN profile |
| **Core LXC (100)** | Pi-hole v6 (Debian LXC) | Local DNS Resolver (`192.168.1.106`) |
| **Docker Host (VM 101)**| Ubuntu 24.04 LTS VM | Docker Engine & Compose Host (`192.168.1.107`) |

---

## Repository Directory Structure

```text
.
├── README.md                           # Master Architecture Documentation
├── LICENSE                             # Open-source License
├── docs/
│   └── images/
│       ├── datanest-architecture-no-ip.png    # High-level Architecture Diagram
│       ├── datanest-architecture-with-ip.png  # Detailed Technical Diagram with IPs
│       └── screenshots/                       # System Verification Screenshots
│           ├── 01-proxmox-dashboard.png
│           ├── 02-pihole-dnsmasq.png
│           ├── 03-npm-ssl-certificates.png
│           ├── 04-ghost-cloudflare-tunnel.png
│           ├── 05-cloudflare-zero-trust-otp.png
│           ├── 06-tailscale-admin-routes.png
│           └── 07-immich-dashboard.png
├── configs/
│   ├── pihole/
│   │   └── 04-datanest-homelab.conf    # Pi-hole v6 Custom DNS Routes
│   ├── samba/
│   │   └── smb.conf                    # Samba SMB Storage Configuration
│   └── sysctl/
│       └── 99-tailscale.conf           # Kernel IP Forwarding Parameters
└── docker/
    ├── ghost-stack/
    │   └── docker-compose.yml          # Ghost CMS, MariaDB 10.11, Cloudflared
    ├── nextcloud-stack/
    │   └── docker-compose.yml          # Nextcloud HA, PostgreSQL, Redis
    └── saas-stack/
        └── docker-compose.yml          # Vaultwarden, Stirling PDF, Immich Stack
```

---

## Quick Start & Deployment Playbook

### 1. Enable Kernel IP Forwarding & Docker Subnet Routing
On the Docker Host VM (`192.168.1.107`), enable kernel packet forwarding and inject the `iptables` rule to allow Tailscale WireGuard traffic into Docker bridge networks:

```bash
# Enable IP Forwarding
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf

# Advertise Subnet Route on Tailscale
sudo tailscale up --advertise-routes=192.168.1.0/24 --accept-dns=false

# Fix Docker iptables Connection Timeout Conflict
sudo iptables -I DOCKER-USER -i tailscale0 -j ACCEPT
sudo apt install iptables-persistent -y
sudo netfilter-persistent save
```

### 2. Configure Pi-hole v6 Split DNS
Enable custom configuration parsing in Pi-hole v6 FTL and apply targeted subdomain routes:

```bash
# Enable /etc/dnsmasq.d in Pi-hole v6
pihole-FTL --config misc.etc_dnsmasq_d true

# Create targeted routing file
cat <<EOF | sudo tee /etc/dnsmasq.d/04-datanest-homelab.conf
address=/vault.datanest.my.id/192.168.1.107
address=/portainer.datanest.my.id/192.168.1.107
address=/media.datanest.my.id/192.168.1.107
address=/proxmox.datanest.my.id/192.168.1.105
EOF

# Restart DNS Engine
sudo systemctl restart pihole-FTL
```

### 3. Deploy Production Stacks
Navigate to each directory in `docker/` and deploy the stack:

```bash
# Deploy Public Web Stack (Ghost + MariaDB + Cloudflared)
cd docker/ghost-stack
docker compose up -d

# Deploy SaaS Stack (Vaultwarden + Stirling PDF + Immich)
cd ../saas-stack
docker compose up -d

# Deploy Nextcloud HA Stack
cd ../nextcloud-stack
docker compose up -d
```

---

## 🛠️ Major Case Studies & Troubleshooting Log

1. **Live LVM Disk Expansion**: Expanded the VM root partition live from 21GB to 50GB (`lvextend -l +100%FREE` & `resize2fs`) when database layers exhausted root storage, avoiding `Exit Code 139` crashes without VM downtime.
2. **Cellular IPv6 DNS Override Fix**: Resolved Windows client `NXDOMAIN` issues caused by Telkomsel 4G router forcing IPv6 DNS servers by setting a custom IPv4-only APN profile (`TelkomselIPv4`, PDP: IPv4).
3. **Ghost CMS `ECONNREFUSED` Fix**: Resolved `502 Bad Gateway` caused by Docker startup race conditions by setting health checks on MariaDB 10.11 and mapping `database__connection__host` to the service name `ghost_db`.
4. **Apple iCloud Private Relay Interference**: Solved iOS Safari connection timeouts by disabling Private Relay and "Limit IP Address Tracking" to prevent WireGuard tunnel hijacking.

---

## License & Acknowledgments

Distributed under the **MIT License**. Built with passion as part of the **DataNest Infrastructure Series**.
* Author: **DataNest Lab Team**
* Website & Blog: [https://datanest.my.id](https://datanest.my.id)
