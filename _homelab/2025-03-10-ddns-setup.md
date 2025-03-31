---
title: "Dynamic DNS (DDNS) Setup – Home Lab"
date: 2025-03-10
tags: [ddns, networking, no-ip, debian, firewall, vpn]
author: Gage Neumaier
layout: project
excerpt: "Configured No-IP Dynamic DNS to expose VPN and game servers from a Debian 12 home lab with secure firewall rules and port forwarding."
image:
  path: /assets/homelab/ddns/ddns-logo.png
  srcset:
    1920w: /assets/homelab/ddns/ddns-logo.png
    960w: /assets/homelab/ddns/ddns-logo@0.5x.png
    480w: /assets/homelab/ddns/ddns-logo@0.25x.png
---

## Scenario: Public Access to Select Home Lab Services

**Server OS:** Debian 12  
**Client OS:** Windows 11  
**Purpose:** Use Dynamic DNS to assign a public hostname to the home server, exposing only VPN and game servers through secure port forwarding

In this project, I configured No-IP's Dynamic DNS client on my Debian 12 server. The setup allows remote clients to access VPN and game services over a public domain name, with strict UFW rules and NAT port forwarding for security.

---

## Project Goals

- Register and configure a No-IP hostname
- Set up DDNS client on Debian server
- Maintain updated public IP address with provider
- Configure router for port forwarding (VPN, game servers)
- Lock down access using UFW and static internal IP

---

## Environment Setup

### Server

- **OS:** Debian 12 (headless)
- **Firewall:** UFW enabled
- **Static LAN IP:** Assigned for reliable port forwarding
- **Services Exposed via DDNS:** WireGuard VPN, Game Servers (e.g., Valheim, Minecraft)

### Router

- Port forwarding configured:
  - VPN (UDP 51820)
  - Game servers (e.g., UDP 2456–2458, TCP 25565)

---

## Step 1 – Register No-IP Hostname

Created account at [No-IP](https://www.noip.com/) and registered free hostname:  
`example.ddns.net`

---

## Step 2 – Install No-IP DDNS Client

Downloaded and installed No-IP's Dynamic Update Client (DUC):

```bash
sudo apt install build-essential
cd /usr/local/src
sudo wget http://www.no-ip.com/client/linux/noip-duc-linux.tar.gz
sudo tar xf noip-duc-linux.tar.gz
cd noip-2.1.9-1
sudo make
sudo make install
```

During install, entered No-IP username, password, and selected hostname.

---

## Step 3 – Configure DDNS Client Persistence

Created systemd service for automatic startup (example `noip2.service`):

```ini
[Unit]
Description=No-IP Dynamic DNS Update Client
After=network.target

[Service]
ExecStart=/usr/local/bin/noip2
Restart=always

[Install]
WantedBy=multi-user.target
```

Enabled and started the service:

```bash
sudo systemctl enable noip2
sudo systemctl start noip2
```

---

## Step 4 – Configure UFW for DDNS Services

Allowed only necessary services:

```bash
# VPN
sudo ufw allow 51820/udp

# Valheim
sudo ufw allow 2456:2458/udp

# Minecraft
sudo ufw allow 25565/tcp
```

All other traffic denied by default policy.

---

## Step 5 – Port Forwarding on Router

Mapped external ports to internal static IP of Debian server:
- 51820 UDP → VPN
- 2456–2458 UDP → Valheim
- 25565 TCP → Minecraft

Tested externally using public IP and DDNS domain.

---

## Final Testing

- Verified DDNS domain resolves to public IP
- Confirmed VPN and game server access using domain name
- Ensured UFW limits traffic only to allowed ports
- Validated `noip2` client auto-starts and updates IP on reboot

---

## Deployment Summary

Deployment Log – March 10, 2025

No-IP DDNS successfully configured for home lab access. 

Port forwarding and firewall limited exposure to only necessary services. 

Deployed by: Gage Neumaier

Role: Home Lab Admin / IT Support 
 
Time to Complete: ~45 Minutes
