---
title: "Game Server Hosting – Home Lab"
date: 2024-11-30
tags: [games, debian, docker, firewall, port-forwarding, ddns]
author: Gage Neumaier
layout: project
excerpt: "Hosting multiplayer game servers on a Debian 12 home lab machine using Docker and firewall rules, with external access via DDNS and port forwarding."
image:
  path: /assets/homelab/game-server/valheim-logo.png
  srcset:
    1920w: /assets/homelab/game-server/valheim-logo.png
    960w: /assets/homelab/game-server/valheim-logo@0.5x.png
    480w: /assets/homelab/game-server/valheim-logo@0.25x.png
---

## Scenario: Hosting Multiplayer Games from My Home Lab
Server OS: Debian 12
Client OS: Windows 11
Purpose: Host private and public multiplayer game servers with secure, reliable external access

In this project, I deployed two multiplayer game servers from my Debian 12 home lab. These servers were initially exposed using Playit.gg tunnels and later transitioned to traditional port forwarding with DDNS for stable remote access.

---

## Project Goals
- Host dedicated game servers in isolated environments
- Allow secure external access without exposing the entire network
- Configure DDNS for consistent domain access
- Implement strict firewall rules with ufw

---

## Environment Setup
Games Hosted:
- Valheim (Docker)
- Bedrock Minecraft (Docker)

Port Requirements:
- Valheim: UDP 2456–2458
- Minecraft: TCP 25565

---

## Step 1 – Game Server Deployment
- Minecraft via Docker Compose image itzg/minecraft-server
- Valheim via Docker Compose image lloesche/valheim-server
- Mounted persistent volumes for world saves and configs

### Minecraft
Created Folder for Minecraft:

```bash
mkdir minecraft-server
cd minecraft-server
```

Created Directories for Volume Mounts:

```bash
mkdir data
```

Saved following script under a docker.compose.yml
```yml
services:
  minecraft-bedrock:
    image: itzg/minecraft-bedrock-server
    container_name: mc-bedrock
    restart: unless-stopped
    environment:
      EULA: "TRUE"
      GAMEMODE: "survival"
      DIFFICULTY: "easy"
      SERVER_NAME: "Expired Leftovers"
      MAX_PLAYERS: "5"
    ports:
      - "19132:19132/udp"
    volumes:
      - ./data:/data
    deploy:
      resources:
        limits:
          cpus: "1.5"  # Limits to 1.5 CPU cores
          memory: "2GB"  # Limits to 2GB of RAM
        reservations:
          cpus: "1.0"  # Reserves at least 1 CPU core
          memory: "1.5GB"  # Reserves at least 1.5GB RAM
```
added deploy limitations to avoid resource consumption
{:.note}

Started Server:
```bash
sudo docker compose up -d
```

### Valheim
Created Folder for valheim:

```bash
mkdir vanilla-valheim
cd vanilla-valheim
```

Created Directories for Volume Mounts:

```bash
mkdir config
mkdir data
```

Saved following script under a docker.compose.yml
```yml
services:
  valheim:
    image: lloesche/valheim-server
    container_name: vanilla-valheim
    restart: unless-stopped
    ports:
      - "2456-2457:2456-2457/udp"
    volumes:
      - /home/gage/vanilla-valheim/config:/config
      - /home/gage/vanilla-valheim/data:/opt/valheim
    environment:
      - SERVER_NAME=Midgard  # Change to your desired server name
      - WORLD_NAME=SKyIsland      # Change to your world name
      - WORLD_SEED=uGmGa46WXA   # Set world seed
      - SERVER_PASS=Loki!       # Change to your server password
      - SERVER_PUBLIC=true       # Set to false for a private server
      - UPDATE_INTERVAL=10800    # Auto-update every 3 hours
      - TZ=UTC                   # Set timezone (optional)
      - ADMINLIST_IDS='76561198081425003'  # Add administrator Steam ID
      - EVENT_INTERVAL=7200        # Lower frequency of raids
    deploy:
      resources:
        limits:
          memory: 4G  # Limit container to 4GB RAM
          cpus: "2.0"  # Limit to 2 CPU cores
        reservations:
          memory: 2G  # Reserve 2GB of RAM for the container
          cpus: "1.0"  # Reserve 1 CPU core for the container
```
Added deploy limitations to avoid resource consumption
{:.note}

Started Server:
```bash
sudo docker compose up -d
```

---

## Step 2 – Initial Exposure with Playit.gg
Used Playit.gg for NAT traversal and temporary public URLs
- Pros: Easy to set up
- Cons: Less control, dependency on 3rd-party tunnels

---

## Step 3 – Switched to Traditional Port Forwarding + DDNS
- Configured router to forward game ports to the Debian host
- Integrated DDNS provider (e.g., DuckDNS or No-IP)
- Ensured the hostname automatically updates with IP changes.

## Step 4 – Firewall Rules with UFW

Enabled UFW and allowed only the necessary ports:

```bash
sudo ufw allow 2456:2458/udp  # Valheim
sudo ufw allow 25565/tcp     # Minecraft
```

Verified UFW status:

```bash
sudo ufw status numbered
```

---

## Final Testing
- Connected to both game servers using the DDNS domain
- Verified server reachability from external IP using forwarded ports
- Checked firewall settings to confirm only necessary ports were open
- Verified that services persist after a reboot.

---

## Deployment Summary

Deployment Log – November 30, 2024

Game servers deployed and made externally accessible via DDNS and port forwarding.

Firewall secured using UFW with tight access control.

Deployed by: Gage Neumaier

Role: Home Lab Admin / IT Support

Time to Complete: ~1–2 hours