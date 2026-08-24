# Cybersecurity Homelab
## DISCLAIMER
- AI was used to document this git as a tool to remember and refer to what I have learnt.
## Overview

This repository documents the development of my personal cybersecurity and infrastructure homelab, built to gain hands-on experience with virtualisation, Linux administration, networking, containerisation, monitoring, troubleshooting and defensive security.

Rather than following tutorials without understanding them, the aim of this project is to understand **why technologies are used, what problems they solve and how different components work together as a system**.

The lab has developed from a single Ubuntu Server VM into a containerised environment with internal DNS, reverse proxying, monitoring and security controls.

---

## Hardware

* Dell OptiPlex 3040 SFF
* Intel Core i5-6500
* 16 GB RAM
* 256 GB SSD
* TP-Link 5-Port Gigabit Switch

---

## Current Architecture

```text
Dell OptiPlex 3040
        │
        ▼
    Proxmox VE
        │
        ▼
 Ubuntu Server VM
        │
        ▼
      Docker
        │
        ├── Homepage
        ├── Portainer
        ├── Uptime Kuma
        ├── AdGuard Home
        ├── Nginx Proxy Manager
        ├── Docker Socket Proxy
        │
        └── Monitoring Network
              ├── Node Exporter
              ├── cAdvisor
              ├── Prometheus
              └── Grafana
```

The Ubuntu Server host is additionally protected using:

* UFW firewall
* Hardened OpenSSH configuration
* Ed25519 public-key authentication
* Fail2Ban

---

## Technologies

### Virtualisation

* Proxmox VE
* Ubuntu Server 26.04 LTS

### Containerisation

* Docker Engine
* Docker Compose
* Docker bridge networking
* Docker volumes
* Bind mounts

### Networking

* AdGuard Home
* Nginx Proxy Manager
* Internal DNS
* Reverse proxying
* Docker DNS
* Custom Docker networks

### Monitoring

* Prometheus
* Grafana
* Node Exporter
* cAdvisor

### Security

* UFW
* OpenSSH
* Ed25519 SSH keys
* Fail2Ban
* Private Certificate Authority
* TLS certificates

---

## Internal Network Architecture

Internal services use `.home.lab` DNS names instead of requiring IP addresses and port numbers to be remembered.

For example:

```text
Browser
   │
   ▼
homepage.home.lab
   │
   ▼
AdGuard Home
Internal DNS
   │
   ▼
192.168.0.67
   │
   ▼
Nginx Proxy Manager
Reverse Proxy
   │
   ▼
Homepage
```

This helped me understand the separate responsibilities of DNS and reverse proxying.

**DNS answers:**

> Which IP address belongs to this hostname?

**The reverse proxy answers:**

> Which application should receive a request arriving for this hostname?

---

## Monitoring & Observability

I built a monitoring stack using:

```text
Ubuntu Host
     │
     ├──────────────┐
     ▼              ▼
Node Exporter    cAdvisor
Host Metrics     Container Metrics
     │              │
     └──────┬───────┘
            ▼
       Prometheus
            │
            ▼
         Grafana
```

Prometheus collects and stores time-series metrics while Grafana provides dashboards for visualisation.

Current monitoring includes:

* Host CPU utilisation
* Host memory utilisation
* Root filesystem utilisation
* System load
* Container CPU usage
* Container memory usage
* Container network receive/transmit traffic
* Prometheus target availability

---

## Troubleshooting Highlight — Root Filesystem at 96%

One of the most valuable troubleshooting exercises in the project came from the monitoring stack itself.

Grafana showed that the Ubuntu root filesystem had reached approximately **96% utilisation**.

Rather than immediately deleting files, I investigated the underlying cause.

```text
Grafana
   │
   ▼
96% disk utilisation detected
   │
   ▼
df -h
Confirmed filesystem usage
   │
   ▼
docker system df
Investigated Docker storage
   │
   ▼
lsblk / vgs / lvs
Inspected storage architecture
   │
   ▼
Unused LVM capacity discovered
   │
   ▼
Logical volume extended by 10 GB
   │
   ▼
ext4 filesystem resized
   │
   ▼
Disk utilisation reduced to ~58%
```

The 32 GB VM disk was not actually full.

The underlying LVM volume group contained approximately 14.5 GB of unused capacity which had never been allocated to the root logical volume.

I deliberately extended the logical volume by **10 GB rather than consuming all available space**, retaining some capacity for future storage requirements.

This reinforced the importance of diagnosing the underlying system rather than reacting only to the visible symptom.

---

## Security Hardening

The Ubuntu Server has been hardened using multiple layers of security.

### Firewall

Configured UFW using a default-deny inbound policy while permitting required services such as:

* SSH
* DNS
* HTTP
* HTTPS

### SSH

Remote administration was migrated from password authentication to **Ed25519 public-key authentication**.

Additional hardening included:

* Password SSH authentication disabled
* Direct root SSH login disabled
* SSH configuration validated before restarting the service

### Fail2Ban

Fail2Ban monitors SSH authentication events through the systemd journal.

Current SSH jail policy:

```text
5 failed attempts
within 10 minutes
        │
        ▼
Source IP banned
for 1 hour
```

Together these controls provide multiple defensive layers rather than relying on a single security mechanism.

---

## TLS / HTTPS

A private Certificate Authority has been created for the internal `.home.lab` domain.

The project currently includes:

* Private root CA
* Wildcard `*.home.lab` certificate
* Subject Alternative Names
* Windows trust-store configuration

Deployment of the wildcard certificate through Nginx Proxy Manager is currently **in progress**.

Private keys and sensitive certificate material are excluded from the repository.

---

## Key Concepts Learned

Throughout the project I have developed practical understanding of:

* Linux administration
* Virtualisation
* Docker architecture
* Images and containers
* Persistent storage
* Docker networking
* Internal DNS
* Reverse proxies
* Host and container monitoring
* Time-series metrics
* Prometheus scraping
* Grafana dashboards
* Linux filesystems
* LVM
* Firewalls
* SSH authentication
* Brute-force mitigation
* TLS and certificate authorities
* Structured troubleshooting

---

## Troubleshooting Approach

A principle used throughout the project is:

> **Inspect → Understand → Change → Verify**

Rather than immediately searching for a command to fix a problem, I try to determine:

* What component owns the problem?
* What evidence supports the hypothesis?
* Where is the relevant data stored?
* How are the components communicating?
* What would change if a container or service disappeared?
* How can the result be verified after making a change?

---

## Progress

### Foundation

* ✅ Installed Proxmox VE
* ✅ Deployed Ubuntu Server VM
* ✅ Linux administration fundamentals
* ✅ Installed and configured Docker
* ✅ Docker Compose
* ✅ Docker networking
* ✅ Persistent storage

### Homelab Infrastructure

* ✅ Homepage
* ✅ Portainer
* ✅ Uptime Kuma
* ✅ AdGuard Home
* ✅ Nginx Proxy Manager
* ✅ Internal `.home.lab` DNS
* ✅ Reverse proxy routing
* ✅ Docker Socket Proxy

### Monitoring

* ✅ Node Exporter
* ✅ cAdvisor
* ✅ Prometheus
* ✅ Grafana
* ✅ Host monitoring dashboard
* ✅ Container monitoring dashboard

### Security

* ✅ UFW firewall
* ✅ Ed25519 SSH authentication
* ✅ Disabled password SSH login
* ✅ Disabled root SSH login
* ✅ Fail2Ban
* 🟡 Internal HTTPS / TLS
* ⏳ Docker exposed-port hardening
* ⏳ Centralised logging / SIEM
* ⏳ Backup and recovery

---

## Future Development

Planned areas of development include:

* Complete internal HTTPS deployment
* Reduce unnecessary Docker-published ports
* Improve network access restrictions
* Centralised log collection
* SIEM and security monitoring
* Backup and disaster recovery
* Identity and authentication
* Infrastructure automation
* Infrastructure as Code
* Vulnerability management
