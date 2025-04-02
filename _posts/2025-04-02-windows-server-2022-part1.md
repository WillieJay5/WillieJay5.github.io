---
title: "Windows Server 2022 Eval Lab – Part 1"
subtitle: "Deploying and configuring a Windows Server 2022 VM in preparation for Active Directory"
author: [Gage Neumaier]
date: 2025-04-02
updated: 2025-04-02
layout: post
categories: [help-desk, windows]
tags: [windows-server, active-directory, vmware, home-lab, help-desk, gpo]
excerpt: >
  Step-by-step walkthrough for setting up a Windows Server 2022 Evaluation VM using VMware Workstation Pro. This lab sets the foundation for building an Active Directory domain controller with GPO support for help desk training.
image: 
  path: /assets/img/windows.jpg
  srcset:
    1920w: /assets/img/windows.jpg
    960w: /assets/img/windows@0.5x.jpg
    480w: /assets/img/windows@0.25x.jpg
featured: true
---


# Windows Server 2022 Evaluation Lab (Part 1)

## Goal
Deploy a Windows Server 2022 Evaluation virtual machine using VMware Workstation Pro and configure it to prepare for promotion to a Domain Controller. This is the foundation for a help desk and GPO-focused Active Directory lab.

---

## Tools Used
- **Host OS:** Windows 10/11
- **Virtualization Platform:** VMware Workstation Pro
- **Server OS:** Windows Server 2022 Evaluation (Desktop Experience)
- **Download Link:** [Microsoft Eval Center](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2022)

---

## Virtual Machine Setup (VMware Workstation Pro)

### Step 1: Create a New Virtual Machine
- Launch VMware Workstation Pro
- Select **Create a New Virtual Machine** → Choose **Custom (Advanced)**
- Hardware Compatibility: *Workstation 17.x*

### Step 2: OS and ISO Selection
- Choose: **"I will install the operating system later"** *(avoids Easy Install issues)*
- Guest OS: **Microsoft Windows → Windows Server 2022**

### Step 3: VM Configuration
- CPU: **2 cores** minimum
- Memory: **4–6 GB** (8 GB+ ideal)
- Network Type: **NAT** or **Host-only** for isolated lab
- Disk:
  - Disk Type: **SCSI (Recommended)**
  - Size: **60–80 GB**, single file
  - Optionally use SATA if preferred

  <figure>
    <img src="/assets/apex-design-co/server-deployment/vm-settings.JPG" alt="Virtual Machine Settings">
    <figcaption>Workstation Settings for Deployment</figcaption>
  </figure>

### Step 4: Attach the ISO
- After VM is created, go to **VM Settings → CD/DVD (SATA)**
- Attach the Windows Server 2022 Eval ISO
- Check **Connect at power on**

---

## Windows Server Installation

### Step 1: Boot from ISO
Start the VM, it will boot into the Windows Setup screen.

### Step 2: Select OS Version
Choose:
- **Windows Server 2022 Standard (Desktop Experience)**

Make sure to select Desktop Experience for GUI functionality
{:.note}

<figure>
  <img src="/assets/apex-design-co/server-deployment/os-selection.JPG" alt="Operating System Selection">
  <figcaption>Selection of Operating System</figcaption>
</figure>

### Step 3: Accept License Terms
Accept the Microsoft Software License Terms → Click **Next**

### Step 4: Choose Installation Type
- Choose **Custom: Install Windows only (advanced)**

### Step 5: Partition Setup
- Select the unallocated virtual drive
- Click **New** → Accept default size → **Next**

### Step 6: Let Setup Complete
- Installation will proceed and reboot a few times
- Eventually you'll be prompted to set the **Administrator password**
- Set a strong password for this

### Step 7: First Login
- Log in with `Administrator`
- You’ll land on the Windows Server desktop with **Server Manager** open

<figure>
  <img src="/assets/apex-design-co/server-deployment/post-installation.JPG" alt="GUI Post-Deployment">
  <figcaption>System screen after OS Deployment</figcaption>
</figure>

---

## Post-Installation Configuration

### Step 1: Set a Static IP Address
1. Open **Control Panel → Network and Sharing Center**
2. Go to **Change adapter settings**
3. Right-click Ethernet → **Properties**
4. Double-click **Internet Protocol Version 4 (TCP/IPv4)**
5. Set:

```
IP Address:      192.168.100.10
Subnet Mask:     255.255.255.0
Default Gateway: 192.168.100.1
Preferred DNS:   192.168.100.10  # Use the server's own IP
```
<figure>
  <img src="/assets/apex-design-co/server-deployment/ip-setup.JPG" alt="IP Setup">
  <figcaption>IP Address Configuration</figcaption>
</figure>

### Step 2: Rename the Server
#### Option A: GUI
- Server Manager → *Local Server* → Click on the computer name → Change to `DC01`

<figure>
  <img src="/assets/apex-design-co/server-deployment/computer-name-change.JPG" alt="Changing Computer Name">
  <figcaption>Workstation Settings for Deployment</figcaption>
</figure>

#### Option B: PowerShell
```powershell
Rename-Computer -NewName "DC01" -Restart
```

### Step 3: Install Windows Updates
#### Using `sconfig`
```powershell
sconfig
```
- Option `6`: Install updates
- Option `3`: All feature updates
- Reboot if needed

I ran into errors with this, may be due to it being a Windows Server 2022 Evaluation Edition. Will have to do more research. 
{:.note}

### Step 4: Install VMware Tools
- In VMware: **VM → Install VMware Tools**
- Inside the VM, open the mounted CD → Run `setup64.exe`
- Complete the wizard and reboot

---

## Final Checkpoint Before Domain Controller Setup
At this point, your VM should:
- Be named (e.g., `DC01`)
- Have a static IP address (e.g., `192.168.100.10`)
- Be up to date
- Have VMware Tools installed

---

## Next Steps (Coming in Part 2)
- Install AD DS and promote the server to a Domain Controller
- Create OUs, users, and groups in Active Directory
- Configure Group Policy Objects (GPOs)
- Join client machines to the domain
- Simulate help desk tasks

Stay tuned!
