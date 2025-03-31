---
title: "Plex Media Server – Home Lab"
date: 2025-02-17
tags: [plex, docker, debian, media, ufw]
author: Gage Neumaier
layout: project
excerpt: "Deployed a Plex Media Server on Debian 12 using Docker Compose. Media files are hosted locally and access is restricted to LAN using UFW."
image:
  path: /assets/homelab/plex/plex-logo.jpg
  srcset:
    1920w: /assets/homelab/plex/plex-logo.jpg
    960w: /assets/homelab/plex/plex-logo@0.5x.jpg
    480w: /assets/homelab/plex/plex-logo@0.25x.jpg
---

## Scenario: Local Media Streaming in the Home Lab

**Server OS:** Debian 12  
**Client OS:** Windows 11  
**Purpose:** Stream personal media library over the LAN using Plex Media Server in a containerized environment

In this project, I deployed Plex Media Server using Docker Compose on my Debian 12 home lab server. Media files are stored in a local directory and access is restricted to LAN-only via UFW.

---

## Project Goals

- Deploy Plex Media Server using Docker Compose
- Configure persistent media volume mapping
- Restrict access using UFW to local network clients
- Ensure Plex auto-starts and remains persistent across reboots

---

## Environment Setup

### Server

- **OS:** Debian 12 (headless)
- **Media Directory:** `/home/gage/Plex`
- **Firewall:** UFW enabled
- **Docker Runtime:** Docker + Docker Compose

### Client

- **OS:** Windows 11
- **Access Method:** Web browser via `http://<server-ip>:32400/web`

---

## Step 1 – Define Docker Compose File

Saved as `docker-compose.yml`:

```yaml
version: "3.8"
services:
  plex:
    image: linuxserver/plex
    container_name: plex
    network_mode: host
    environment:
      - PUID=1000
      - PGID=1000
      - VERSION=docker
    volumes:
      - /home/gage/Plex:/config
      - /home/gage/Plex/Media:/media
    restart: unless-stopped
```
Uses `network_mode: host` for better DLNA/local discovery
{:.note}

---

## Step 2 – Launch Plex Container

```bash
docker compose up -d
```

Plex UI accessible from browser:  
`http://<server-ip>:32400/web`

---

## Step 3 – Configure UFW Firewall Rules

Allowed only LAN clients (example subnet: 192.168.1.0/24):

```bash
sudo ufw allow from 192.168.1.0/24 to any port 32400 proto tcp
```

Then enable UFW (if not already):

```bash
sudo ufw enable
```

---

## Step 4 – Final Configuration in Plex UI

1. Create Plex account (or sign in)
2. Add `/media` directory as library root
3. Configure library auto-scanning
4. Set “Allow LAN only” for network access (in Plex settings)

---

## Final Testing

- Verified access to Plex from Windows browser using local IP and port
- Confirmed LAN-only traffic via UFW rules
- Validated media availability, playback, and folder structure
- Ensured Plex container autostarts and config persists after reboot

---

## Deployment Summary

Deployment Log – Feburary 17, 2025

Plex Media Server deployed with Docker Compose.

Access restricted to local network via UFW.

Persistent media storage set up with mounted host volumes.

Deployed by: Gage Neumaier

Role: Home Lab Admin / IT Support

Time to Complete: ~30 Minutes
