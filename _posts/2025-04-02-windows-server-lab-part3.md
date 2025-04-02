---
title: "Windows Server 2022 Eval Lab – Part 3"
subtitle: "Creating and Securing File Shares with Active Directory Groups"
author: [Gage Neumaier]
date: 2025-04-02
updated: 2025-04-02
layout: post
categories: [help-desk, windows]
tags: [windows-server, active-directory, file-sharing, ntfs-permissions, gpo, help-desk-lab]
excerpt: >
  In Part 3 of this Windows Server lab series, we configure secure department-based file shares using NTFS and share permissions, and map drives via Group Policy. This setup reinforces access control using AD security groups in a help desk lab environment.
image: 
  path: /assets/img/windows.jpg
  srcset:
    1920w: /assets/img/windows.jpg
    960w: /assets/img/windows@0.5x.jpg
    480w: /assets/img/windows@0.25x.jpg
---

# Windows Server 2022 Evaluation Lab (Part 3)

## Goal
Set up department-specific file shares on the domain controller (or file server) and enforce access control using NTFS and Share permissions, aligned with Active Directory group memberships.

---

## Step 1: Create Department Folders

1. On your domain controller (or a dedicated file server), create a shared folder structure:

   ```
   C:\Shares\HR
   C:\Shares\IT
   C:\Shares\Finance
   ```
2. Right-click each folder → **Properties → Sharing → Advanced Sharing**
   - Check **Share this folder**
   - Set Share Name (e.g., `HRShared`)
   - Click **Permissions**:
     - Remove `Everyone`
     - Add the relevant AD group (e.g., `HRGroup`) and grant **Read** or **Change** as needed

<figure>
  <img src="/assets/apex-design-co/file-sharing/hr-drive-permissions.JPG" alt="HR File Sharing Permissions">
  <figcaption>Permissions for the HR Group for file sharing</figcaption>
</figure>

## Step 2: Set NTFS Permissions

1. Go to **Properties → Security tab** of each folder
2. Remove or restrict default entries like `Everyone` or `Authenticated Users`
3. Add:
   - `HRGroup` → Grant **Modify** or **Read**
   - Repeat for each department and folder

> Note: NTFS permissions control actual access. Share permissions act as a filter — **NTFS wins** when more restrictive.

<figure>
  <img src="/assets/apex-design-co/file-sharing/hr-drive-ntfs-permissions.JPG" alt="HR Drive NTFS Permissions">
  <figcaption>Permissions for HR Group in NTFS</figcaption>
</figure>

---

## Step 3: Map Network Drives via GPO

1. Open **Group Policy Management**
2. Create a new GPO (e.g., `Map HR Drive`) and link it to the relevant OU (e.g., `HR`)
3. Edit the GPO:
   - Navigate to:
     `User Configuration → Preferences → Windows Settings → Drive Maps`
   - Right-click → **New → Mapped Drive**
     - Location: `\\DC01\HRShared`
     - Drive Letter: `H:`
     - Label: `HR Drive`
     - Action: **Create**
4. Under the **Common** tab:
   - Check **Item-level targeting**
   - Add a security group condition for `HRGroup`

> Tip: Use one GPO per department for clean organization and targeted access.

---

## Step 4: Test Access from Client

1. Log in to the domain as a user from a department (e.g., `corp\j.doe` from `HRGroup`)
2. Confirm that:
   - The network drive maps automatically
   - Access matches group permissions
   - Users outside the group receive an error or no drive

> Tip: You can use `net use` in Command Prompt to manually test drive mappings.

---

## Optional: Enable Auditing (ISO 27001 A.12.4.1)

To track file access:
1. Enable object access auditing via GPO:
   - `Computer Configuration → Policies → Windows Settings → Security Settings → Local Policies → Audit Policy`
   - Enable **Audit object access** (Success/Failure)
2. On the shared folder:
   - Right-click → **Properties → Security → Advanced → Auditing**
   - Add an auditing entry for the relevant group

> Note: This supports compliance with ISO 27001 control A.12.4.1 for monitoring file access.

---
