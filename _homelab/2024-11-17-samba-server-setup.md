---
title: "Samba File Server Setup – Home Lab"
date: 2024-11-17
tags: [linux, samba, file-sharing, debian, ufw]
author: Gage Neumaier
layout: project
excerpt: "Configuration of a Samba server on Debian 12 to provide secure file sharing between a Linux server and Windows clients in a home lab environment."
image:
  path: /assets/homelab/samba/samba-logo.png
  srcset:
    1920w: /assets/homelab/samba/samba-logo.png
    960w: /assets/homelab/samba/samba-logo@0.5x.png
    480w: /assets/homelab/samba/samba-logo@0.25x.png
---

## Scenario: Home Lab File Sharing

**Server OS:** Debian 12  
**Client OS:** Windows 11  
**Purpose:** LAN file sharing for local access between a headless server and Windows machines

In this project, I configured Samba on my Debian 12 server to enable Windows users to access shared directories over the LAN. The goal was to support seamless file sharing while locking down access to local traffic only.

---

## Project Goals

- Install and configure Samba on a Debian 12 server
- Set up file shares with the correct permissions
- Ensure only authorized users have access
- Use UFW to limit access to LAN clients
- Confirm access from a Windows 11 machine

---

## Environment Setup

### Server

- **OS:** Debian 12 (headless)
- **Access:** SSH
- **Other Services Running:** Plex, game servers, Samba, VPN, SSH

### Client

- **OS:** Windows 11
- **Access Method:** File Explorer via `\\<server-ip>\<share-name>`

---

## Step 1 – Install and Configure Samba

```bash
sudo apt update
sudo apt install samba
```

### Configuration

Edit `/etc/samba/smb.conf` to add a custom share:

```ini
[Home Lab]
path = /home/gage
writable = yes
browseable = yes
guest ok = no
valid users = gage #make sure to match your samba user
```

Create system user and set Samba password:

```bash
sudo adduser gage #or whaterver username you want
sudo smbpasswd -a gage
```

Set folder permissions:

```bash
sudo chown gage:gage /home/gage
sudo chmod 770 /home/gage
```

---

## Step 2 - Configure UFW Firewall

UFW was used to restrict Samba access to LAN traffic only.

```bash
sudo ufw allow from 192.168.1.0/24 to any app Samba
sudo ufw enable
```

---

## Step 3 - Test Access from Windows Client

On the Windows client:

1. Press `Win + R` → type `\\<server-ip>\Home Lab`  
2. Enter Samba credentials (`gage`)  
3. Confirm read/write access

---

## Final Testing

- Verified UFW rules block external traffic  
- Confirmed access from Windows client with correct credentials  
- Ensured read/write access to the shared folder  
- All configurations persistent after reboot

---

## Deployment Summary

Deployment Log – November 17, 2024  
Samba file server configured and tested in a home lab environment.  
LAN-only access enforced using UFW.  
Deployed by: Gage Neumaier  
Role: Home Lab Admin / IT Support  
Time to Complete: ~30 Minutes
