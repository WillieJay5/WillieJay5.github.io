---
title: "SSH Key Authentication – Home Lab"
date: 2025-01-10
tags: [ssh, debian, security, linux, openssh]
author: Gage Neumaier
layout: project
excerpt: "Configured SSH key-based authentication on a Debian 12 server to enhance remote access security from a Windows client. Password login was disabled after successful setup."
image:
  path: /assets/homelab/ssh/ssh-logo.png
  srcset:
    1920w: /assets/homelab/ssh/ssh-logo.png
    960w: /assets/homelab/ssh/ssh-logo@0.5x.png
    480w: /assets/homelab/ssh/ssh-logo@0.25x.png
---

## Scenario: Securing Remote Access to Home Server

**Server OS:** Debian 12  
**Client OS:** Windows 11  
**Purpose:** Replace password-based SSH authentication with a secure key pair

In this project, I configured SSH key authentication to connect securely from a Windows client to a headless Debian 12 server. The default password-based login was disabled after confirming successful key-based access.

---

## Project Goals

- Generate an SSH key pair from a Windows client
- Set up the public key on the Debian server
- Test key-based login functionality
- Harden `sshd_config` and disable password login

---

## Environment Setup

### Server

- **OS:** Debian 12 (headless)
- **Access:** SSH
- **Other Services Running:** Plex, game servers, Samba, VPN, SSH

### Client

- **OS:** Windows 11
- **Access Method:** OpenSSH via PowerShell / Windows Terminal

---

## Step 1 – Generate SSH Key Pair

On the Windows client:

```bash
ssh-keygen -t ed25519 -C "my_email@example.com"
```
- Saved to: C:\Users\<username>\.ssh\id_ed25519
- Passphrase set for added protection

---

## Step 2 – Add Public Key to Debian Server

Using ssh-copy-id (or manually):

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@<server-ip>
```
Or:
```bash
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh user@<server-ip> "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

---

## Step 3 - Harden SSH Daemon

Edit /etc/ssh/sshd_config on the server:
```conf
PasswordAuthentication no
PermitRootLogin no
PubkeyAuthentication yes
```
Then restart SSH:
```bash
sudo systemctl restart ssh
```

---

## Step 4 – Test Key-Based Login

On the Windows client:
```bash
ssh user@<server-ip>
```
Login should succeed without prompting for password

---


## Final Testing
- Confirmed password login is disabled
- Verified successful key-based access from Windows 11
- Ensured configuration survives reboot
- Validated that root login is blocked

---

## Deployment Summary
Deployment Log – January 10, 2025

SSH key-based login successfully deployed in a home lab.

Password authentication fully disabled after testing.

Deployed by: Gage Neumaier

Role: Home Lab Admin / IT Support

Time to Complete: ~20 Minutes


