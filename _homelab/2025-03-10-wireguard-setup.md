---
title: "VPN Setup – Home Lab"
date: 2025-03-10
tags: [vpn, wireguard, debian, networking, security]
author: Gage Neumaier
layout: project
excerpt: "Configured a WireGuard VPN server on Debian 12 to securely access home lab services such as SSH and Samba from external networks."
image: /assets/homelab/wireguard/wireguard-logo.svg
featured: true
---

## Scenario: Secure Remote Access to Home Lab

**Server OS:** Debian 12  
**Client OS:** Windows 11  
**Purpose:** Create a secure, private tunnel to access internal services (SSH, Samba, etc.) from anywhere

In this project, I set up a WireGuard VPN server on my Debian 12 machine to provide encrypted remote access to my home lab. The VPN allows me to securely reach internal services like SSH and file shares without exposing them directly to the internet.

---

## Project Goals

- Install and configure WireGuard on Debian 12
- Create peer configuration for a Windows 11 client
- Allow access to internal services through the VPN
- Secure traffic with firewall rules

---

## Environment Setup

### Server

- **OS:** Debian 12 (headless)
- **VPN Software:** WireGuard
- **Network Interface:** `wg0`
- **Services Accessible via VPN:** SSH, Samba

### Client

- **OS:** Windows 11
- **Client Tool:** WireGuard for Windows

---

## Step 1 – Install WireGuard on Debian Server

```bash
sudo apt update
sudo apt install wireguard
```

---

## Step 2 – Create Key Pairs

```bash
wg genkey | tee privatekey | wg pubkey > publickey
```

Stored keys in `/etc/wireguard/` with strict permissions:

```bash
sudo chmod 600 /etc/wireguard/privatekey
```

---

## Step 3 – Configure WireGuard Server

Created `/etc/wireguard/wg0.conf`:

```ini
[Interface]
PrivateKey = <server-private-key>
Address = 10.0.0.1/24
ListenPort = 51820

PostUp = ufw route allow in on wg0 out on eth0
PostUp = iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = ufw route delete allow in on wg0 out on eth0
PostDown = iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
```

Started and enabled the service:

```bash
sudo systemctl start wg-quick@wg0
sudo systemctl enable wg-quick@wg0
```

---

## Step 4 – Configure Windows Peer

Used WireGuard GUI on Windows:

```ini
[Interface]
PrivateKey = <client-private-key>
Address = 10.0.0.2/24
DNS = 10.0.0.1

[Peer]
PublicKey = <server-public-key>
Endpoint = <my-ddns-domain>:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

Imported configuration and connected using the GUI.

---

## Step 5 – Update UFW Firewall Rules

```bash
sudo ufw allow 51820/udp
sudo ufw allow from 10.0.0.0/24
```

Enabled IP forwarding in `/etc/sysctl.conf`:

```bash
net.ipv4.ip_forward=1
```

Applied with:

```bash
sudo sysctl -p
```

---

## Final Testing

- Confirmed client can connect from external Wi-Fi
- Verified internal IP reachability (e.g., `ssh 10.0.0.1`, `\10.0.0.1\Home Lab`)
- Validated firewall allows only intended VPN traffic
- Confirmed persistent service across reboots

---

## Deployment Summary

Deployment Log – March 10, 2025  

WireGuard VPN successfully configured on Debian 12.  

Enabled secure external access to internal services with full encryption.  

Deployed by: Gage Neumaier  

Role: Home Lab Admin / IT Support  

Time to Complete: ~45 Minutes

