---
title: "Windows Workstation Setup – Apex Design Co."
date: 2025-03-24
tags: [it-support, help-desk, windows, chocolatey, virtual-machine]
author: Gage Neumaier
layout: project 
excerpt: "End-to-end deployment of a Windows virtual machine, simulating a Help Desk scenario at a small design company."
image: 
  path: /assets/img/windows.jpg
  srcset:
    1920w: /assets/img/windows.jpg
    960w: /assets/img/windows@0.5x.jpg
    480w: /assets/img/windows@0.25x.jpg
featured: false
---

## Scenario: Apex Design Co. Onboarding

**Company:** Apex Design Co.  
**Industry:** Graphic & Web Design  
**Employees:** 15  
**Environment:** Remote-first with limited on-site workstations

Apex is onboarding a new in-office graphic designer, *Jordan Doe*. As the Help Desk Technician, I was assigned to set up a Windows 10 workstation from scratch. This includes OS deployment, standard user creation, software installation, and ensuring the system is configured and ready for Day 1 use.

---

## Project Goals

- Deploy Windows 10 in a virtual lab environment
- Create and configure an administrator and standard user
- Install core company software using PowerShell + Chocolatey
- Apply standard system settings and prepare the system for handoff
- Document the entire process for audit and repeatability

---

## Environment Setup

### Host Machine
- **OS:** Windows 11
- **Virtualization Software:** VMware Workstation Player (Free)

### Guest VM
- **OS:** Windows 10 (64-bit)
- **Specs:** 2 vCPU, 4 GB RAM, 60 GB disk
- **Tools Installed:** VMware Tools

---

## Step 1 – Virtual Machine Deployment

### 1.1 Download Required Files
- Windows 10 ISO from [Microsoft](https://www.microsoft.com/software-download/windows10)

### 1.2 Create the VM in VMware Player
- New VM → Use ISO file
- Guest OS: Microsoft Windows 10 x64
- Name: `APEX-WKS-DESIGN01`
- Store disk as single file (60 GB)
- Customize:
  - 2 vCPUs
  - 4 GB RAM
  - Enable 3D acceleration

<figure>
  <img src="/assets/apex-design-co/deployment/VM-hardware.JPG" alt="Virtual Machine Settings">
  <figcaption>Workstation Settings for Deployment</figcaption>
</figure>

### 1.3 Install Windows
- Follow Windows installer prompts
  - Select "I don't have a product key" (skips activation for lab use)
  - Windows 10 Pro
  - Custom Installation
- Create local admin account: `ApexAdmin`
- Set PC name to: `APEX-WKS-DESIGN01`
- Skip Microsoft account → use Offline Account
- Apply latest updates

<figure>
  <img src="/assets/apex-design-co/deployment/clean-install.JPG" alt="Workstation Setup Screenshot">
  <figcaption>Workstation After Initial Setup</figcaption>
</figure>

---

## Step 2 – User Account Creation

### 2.1 Create Standard User
- Admin Account: `ApexAdmin`
- Standard User: `j.doe`
  - Temp password: `TempPassword123!`

### Steps:
1. Open **Settings → Accounts → Other Users**
2. Add local user → skip Microsoft account
3. Name: `j.doe`
4. Set account type to **Standard User**

<figure>
  <img src="/assets/apex-design-co/deployment/user-account-creation.JPG" alt="User Deployment">
  <figcaption>Showcase of successful user deployment</figcaption>
</figure>

---

## Step 3 – Install VMware Tools (Guest Additions)

To improve performance, resolution scaling, and enable clipboard/file drag-and-drop between host and VM, install VMware Tools.

### 3.1 Launch VMware Tools Installer
- In **VMware Workstation Player**, go to the top menu:
  - `Player → Manage → Install VMware Tools`
- This will mount a virtual CD inside the VM as **Drive D:**

### 3.2 Run the Installer in the VM
- Open **File Explorer**
- Navigate to **D:\**
- Double-click `setup.exe` to launch the VMware Tools installer
- Follow the prompts to complete installation (Typical setup)
- Reboot the VM when prompted

## Step 4 – Install Company Software (via Chocolatey)

### Software List:
- Google Chrome
- Zoom
- Slack
- Notepad++
- 7-Zip

### 4.1 Install Chocolatey
run this script as administrator
~~~powershell
Set-ExecutionPolicy Bypass -Scope Process -Force
[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.SecurityProtocolType]::Tls12
iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))
~~~

install chocolatey
{:.figcaption}

~~~powershell
choco install googlechrome zoom slack notepadplusplus 7zip -y
~~~

run chocolatey to install needed apps
{:.figcaption}

<figure>
  <img src="/assets/apex-design-co/deployment/chocolatey-deployment.JPG" alt="App Installation">
  <figcaption>Successful App Deployment</figcaption>
</figure>

Chrome installation failed so I had to install manually
{:.note}

## Final Testing
- Logged in as `j.doe` to validate user setup
- Verified access to installed apps
- Confirmed lack of admin privileges
- Confirmed smooth desktop experience

## Deployment Summary
Deployment Log - March 24,2025
Workstation `APEX-WKS-DESIGN01` configured and tested for user Jordan Doe.
Applications Installed, user profile created, and VM snapchot saved.
System ready for handoff.
- Deployed by: Gage Neumaier
- Role IT Support Technician
- Time to Complete: ~45 Minutes
