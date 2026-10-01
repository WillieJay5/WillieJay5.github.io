---
layout: page
title: "Homelab Build & Architecture"
description: >
  A detailed breakdown of my homelab infrastructure, hardware compute nodes, network segmentation, and deployed security services.
permalink: /lab-build/
---

This page serves as a living document of my homelab infrastructure, detailing the physical hardware, network architecture, and service deployments that power my self-hosted environment.
{:.lead}[cite: 3, 4]

* toc
{:toc .large-only}[cite: 2, 4, 5]

## Infrastructure Overview

My homelab is built around a hybrid compute and routing topology. The core design principles are **strict network segmentation**, **centralized operational visibility**, and **reproducible infrastructure**.

                  [ ISP Gateway / Internet ]
                               │
                       [ OPNsense Firewall ]
                               │
            ----------------------------------------
            │                                       │
    [ VLAN 10: Management ]                     [ VLAN 30: Lab ]
            │                                       │
     Proxmox VE Node                            Graylog SIEM
    Switch Management                          Cowrie Honeypot


---

## Hardware Specifications

The environment runs on dedicated mini-PCs and custom bare-metal nodes to optimize power consumption while maintaining performance.

| Node Name | Hardware / Form Factor | Primary Role | OS / Hypervisor | Specs |
|:---|:---|:---|:---|:---|
| `pve-node-01` | Custom PC | Primary Compute & VMs | Proxmox VE 8.x | 8 Cores, 16GB RAM, 1TB SSD |
| `router-01` | Bare-Metal x86 Mini-PC | Network Firewall & Routing | OPNsense | 4x 2.5GbE Intel NICs, 16GB RAM |
{:.scroll-table}[cite: 3]

---

## Network & VLAN Segmentation

Network isolation is managed by OPNsense, enforcing strict firewall rules between administrative interfaces, daily workstation traffic, and isolated lab environments.

| VLAN ID | Subnet | Name | Description & Security Policy |
|:---|:---|:---|:---|
| `VLAN 5` | `192.168.5.0/24` | Management | Proxmox hypervisors, switch management, and IPMI interfaces. No outbound internet. |
| `VLAN 100` | `192.168.100.0/24` | Lab | Isolated sandbox for security testing. |
| `VLAN 50` | `192.168.50.0/24` | Regular Network | Day to Day network needs. |
{:.scroll-table}[cite: 3]

---

## Deployed Services & Containers

Most services are virtualized on Proxmox VE using either lightweight LXC containers or dedicated Debian virtual machines.

### Core Operations & Monitoring
*   **Graylog SIEM:** Centralized log aggregation server consuming GELF streams from servers, firewalls, and application containers.
*   **Nginx Reverse Proxy:** Handles SSL/TLS termination using Certbot with Cloudflare DNS challenge validation.

### Lab & Security Testing
*   **Cowrie SSH Honeypot:** Medium-interaction honeypot running on Debian 12 to emulate an SSH shell, logging brute-force attempts directly to Graylog.
