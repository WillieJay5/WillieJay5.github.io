---
title: "UFW Firewall Rules Overview – Home Lab"
date: 2025-03-12
tags: [ufw, firewall, debian, security, networking]
author: Gage Neumaier
layout: project
excerpt: "Hardened Debian 12 home lab server with UFW firewall rules for SSH, VPN, Plex, Samba, and game servers. Configured LAN-only and DDNS-restricted access."
image:
  path: /assets/homelab/firewall/ufw-logo.png
  srcset:
    1920w: /assets/homelab/firewall/ufw-logo.png
    960w: /assets/homelab/firewall/ufw-logo@0.5x.png
    480w: /assets/homelab/firewall/ufw-logo@0.25x.png
---

## Scenario: Network Hardening with UFW

**Server OS:** Debian 12  
**Purpose:** Restrict traffic to only trusted IPs and ports using UFW (Uncomplicated Firewall) on a multi-service home lab server

In this project, I applied and tested UFW firewall rules to protect my Debian 12 home lab. The server hosts multiple services like Plex, Samba, VPN, and game servers. Each service has its own port and access policy based on LAN access or DDNS exposure.

---

## Project Goals

- Default-deny all inbound traffic
- Allow SSH, Samba, and Plex only from the LAN
- Allow VPN and game server ports via DDNS/public IP
- Ensure UFW persists across reboots

---

## Environment Setup

### Network Overview

- **Server IP:** Static internal IP on LAN
- **Client Range:** 192.168.1.0/24
- **DDNS Domain:** Used for VPN and game servers
- **Router:** Port forwards defined only for DDNS-accessible services

---

## Step 1 – Enable UFW with Default Policy

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

---

## Step 2 – Allow Essential Services

```bash
# SSH from LAN
sudo ufw allow from 192.168.1.0/24 to any port 22 proto tcp

# Samba from LAN
sudo ufw allow from 192.168.1.0/24 to any port 445 proto tcp

# Plex from LAN
sudo ufw allow from 192.168.1.0/24 to any port 32400 proto tcp
```

---

## Step 3 – Allow VPN and Game Server Access (Public)

```bash
# WireGuard VPN (No-IP/DDNS domain forwards here)
sudo ufw allow 51820/udp

# Valheim Game Server
sudo ufw allow 2456:2458/udp

# Minecraft Game Server
sudo ufw allow 25565/tcp
```

---

## Step 4 – Enable and Persist UFW

```bash
sudo ufw enable
sudo ufw status numbered
```

Confirmed all rules active. Rebooted server to ensure UFW persisted and didn’t block itself.

---

## Final Testing

- Verified only intended ports respond to `nmap` scans
- Tested Plex and Samba from LAN clients only
- Confirmed game server and VPN access externally via DDNS
- Validated security policy aligns with service exposure

---

## Deployment Summary

Deployment Log – March 12, 2025  
- UFW configured with a hardened rule set to secure Debian 12 home lab.  
- Services limited to LAN or DDNS clients based on role.  
- Deployed by: Gage Neumaier  
- Role: Home Lab Admin / IT Support  
- Time to Complete: ~20 Minutes
