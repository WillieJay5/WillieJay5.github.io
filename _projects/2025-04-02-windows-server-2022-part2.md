---
title: "Windows Server 2022 Eval Lab – Part 2"
subtitle: "Promoting a Server to a Domain Controller and Deploying ISO 27001-Aligned GPOs"
author: [Gage Neumaier]
date: 2025-04-02
updated: 2025-04-02
layout: project
tags: [windows-server, active-directory, gpo, iso27001, domain-controller, help-desk-lab]
excerpt: >
  In this part of the lab series, we promote our Windows Server 2022 VM to a domain controller, build an organizational structure with users and groups, and configure ISO 27001-aligned Group Policy Objects (GPOs) to simulate a secure, help desk-ready AD environment.
image: 
  path: /assets/img/windows.jpg
  srcset:
    1920w: /assets/img/windows.jpg
    960w: /assets/img/windows@0.5x.jpg
    480w: /assets/img/windows@0.25x.jpg
featured: true
---


# Windows Server 2022 Evaluation Lab (Part 2)

## Goal

Promote the configured Windows Server 2022 VM to a Domain Controller and set up a basic Active Directory Domain Services (AD DS) environment with a simulated organizational structure and baseline Group Policy Objects (GPOs).

---

## Prerequisites

From Part 1:

- VM is running Windows Server 2022 (Desktop Experience)
- Server name is set (e.g., `DC01`)
- Static IP address is configured (e.g., `192.168.100.10`)
- VMware Tools and updates are installed

---

## Step 1: Install AD DS and DNS Roles

### Using Server Manager

1. Open **Server Manager** → Click **Manage** → **Add Roles and Features**
2. Role-based or feature-based installation → Select your local server
3. On the **Server Roles** screen:
   - Check **Active Directory Domain Services**
   - DNS Server will auto-select — keep it checked
   - Click **Next** until the install button is available
4. Click **Install** (do not close Server Manager)

Do not reboot yet — we’ll promote to a domain controller next.
{:.note}

<figure>
  <img src="/assets/apex-design-co/AD-DC-promotion/AD-DS-installation.JPG" alt="AD DS Installation">
  <figcaption>AD DS Installation</figcaption>
</figure>



---

## Step 2: Promote to Domain Controller

### After Role Installation

1. Click the **flag icon** in Server Manager → Select **Promote this server to a domain controller**

   <figure>
   <img src="/assets/apex-design-co/AD-DC-promotion/dc-promotion.JPG" alt="DC Flag">
   <figcaption>Flag for DC Promotion</figcaption>
   </figure>
2. Choose **Add a new forest**
   - Root domain name: `corp.local` *(or use your own custom domain)*
3. Set **Directory Services Restore Mode (DSRM)** password
4. Leave DNS and NetBIOS settings default unless customizing
5. Review default file paths → Click **Next**
   <figure>
      <img src="/assets/apex-design-co/AD-DC-promotion/dc-settings.JPG" alt="DC Settings">
      <figcaption>Settings for DC Configuration</figcaption>
   </figure>
6. Click **Install**

The server will promote itself to a DC and automatically reboot when complete.


---

## Step 3: Log in to Your New Domain

After reboot:

- Log in with:

```
Username: corp\Administrator
Password: [your admin password]
```

- `corp` is the NetBIOS domain name derived from `corp.local`

Open **Server Manager** → Confirm AD DS and DNS are listed and healthy

---

## Step 4: Create Organizational Units (OUs), Users, and Groups

### Open Active Directory Users and Computers (ADUC)

- Server Manager → Tools → **Active Directory Users and Computers**

### 1. Create Organizational Units

Right-click the domain root (`corp.local`) → **New → Organizational Unit** Create the following OUs:

- `IT`
- `HR`
- `Finance`

**Warning:** AD already includes default containers called `Users` and `Computers` — you do not need to create a new OU with the same name. Avoid duplicating container names.
{:.note}

### 2. Create Test Users

Within each department OU:

- Right-click the OU → **New → User**
- Fill in user details and set an initial password

Example users:

- `j.doe` in `HR`
- `it.tech` in `IT`
- `a.manager` in `Finance`

Unchecked "User must change password at next logon" for testing. This otherwise would be enabled.
{:.note}

### 3. Create Security Groups

Right-click each department OU → **New → Group** Use **Global / Security** group type

Example groups:

- `HelpDesk` (IT)
- `HRGroup` (HR)
- `FinanceGroup` (Finance)

Add relevant users to their corresponding groups:

- Add `j.doe` to `HRGroup`
- Add `it.tech` to `HelpDesk`
- Add `a.manager` to `FinanceGroup`

If you accidentally create a group or object in the wrong OU and get a permissions error when trying to delete it:

1. Enable **Advanced Features** in ADUC (View → Advanced Features)
2. Right-click the object → **Properties → Object tab**
3. Uncheck **"Protect object from accidental deletion"**
4. You can then delete or move the object

Here's an example of my OU structure:
<figure>
  <img src="/assets/apex-design-co/AD-DC-promotion/users-group-creation.JPG" alt="Users and Group Creation">
</figure>
---

## Step 5: Create and Link Basic GPOs (Aligned with ISO 27001)

### Open Group Policy Management

- Server Manager → Tools → **Group Policy Management**

### Create GPOs at the Domain or OU level

These GPOs are designed to support ISO/IEC 27001 controls, particularly around access control, system security, and user awareness.

#### Enforce Password Policy (ISO 27001 A.9.2.4, A.9.4.3)

- Minimum password length: e.g., 12 characters
- Enforce password history
- Maximum password age: 60-90 days
- Account lockout threshold: e.g., 5 invalid attempts

<figure>
  <img src="/assets/apex-design-co/AD-DC-promotion/password-policy.JPG" alt="Password GPO">
  <figcaption>Settings for password GPO</figcaption>
</figure>

#### Add Logon Banner (ISO 27001 A.8.2.3, A.9.3.1)

Computer Configuration → Policies → Windows Settings → Security Settings → Local Policies → Security Options

- "Interactive logon: Message title/text"

<figure>
  <img src="/assets/apex-design-co/AD-DC-promotion/logon-banner-setup.JPG" alt="Logon Banner Settings">
  <figcaption>Settings for Logon Banner</figcaption>
</figure>

#### Disable Control Panel (ISO 27001 A.9.4.1 – Restriction of Access Rights)

User Configuration → Policies → Administrative Templates → Control Panel → Prohibit access to Control Panel

<figure>
  <img src="/assets/apex-design-co/AD-DC-promotion/control-panel-control.JPG" alt="Control Panel Settings">
  <figcaption>Settings for Disabling Control Panel</figcaption>
</figure>

#### Map Network Drive via GPO (ISO 27001 A.9.2.6 – Removal or Adjustment of Access Rights)

User Configuration → Preferences → Windows Settings → Drive Maps

For individual group control, I created separate policies for each OU and linked them directly to the corresponding group. This creates a shared drive for each OU.
{:.note}

---

## Step 6: Join a Windows Client VM

### Setup

- Spin up a Windows 10/11 VM
- Ensure the VM is on the **same virtual network** as your domain controller (NAT or Host-only in VMware Workstation)
- Set a **unique static IP address** (e.g., `192.168.100.50`) to avoid duplication errors
- Set **Preferred DNS** to the DC IP (e.g., `192.168.100.10`)
- Set static IP in same subnet (e.g., `192.168.100.50`)
- Set **Preferred DNS to the DC IP** (e.g., `192.168.100.10`)

<figure>
  <img src="/assets/apex-design-co/AD-DC-promotion/client-ip-configuration.JPG" alt="Client IP Settings">
  <figcaption>Settings for Client Static IP</figcaption>
</figure>

### Join the Domain

- Right-click **This PC** → Properties → **Rename this PC (Advanced)**
- Click **Change** → Join domain: `corp.local`
- Use domain admin credentials
- Reboot

<figure>
  <img src="/assets/apex-design-co/AD-DC-promotion/client-domain-configuration.JPG" alt="Domain Configuration">
  <figcaption>Settings for Domain</figcaption>
</figure>

I had to reduce the name of the computer since NetBIOS only allows a maximum of 15.
{:.note}

Log in as a domain user (e.g., `corp\j.doe`) to test GPOs and AD functionality

---

## Troubleshooting: Network and Domain Join

If your client cannot reach the domain controller or join the domain, try the following steps:

### Verify IP Configuration (Client)
Open Command Prompt and run:
```cmd
ipconfig /all
```
Check that:
- IPv4 address is in the same subnet as the DC (e.g., `192.168.100.x`)
- Default gateway matches the network
- Preferred DNS is set to the DC’s IP (e.g., `192.168.100.10`)
- No duplicate IP warning or fallback to `169.254.x.x`

<figure>
  <img src="/assets/apex-design-co/AD-DC-promotion/ip-duplicate-troubleshooting.JPG" alt="IP Configuration">
</figure>

My IP was using a duplicate, so I changed it to an unused IP.
{:.note}

### Test Network Connectivity
Use `ping` and `tracert` from the client:
```cmd
ping 192.168.100.10
tracert 192.168.100.10
```
You should get a direct response with no timeouts. If you see "Destination host unreachable," check VMware network settings.

### Test DNS Resolution
From the client:
```cmd
nslookup corp.local
```
This should return the IP of your domain controller. If it times out or fails, DNS isn’t resolving properly.

### Restart Core Services on DC
If DNS is stuck or returning Event ID 4013:
```cmd
net stop netlogon
net stop dns
net start dns
net start netlogon
ipconfig /registerdns
```
Then reboot the domain controller.

---

At this point, you have:

- A Windows Server 2022 domain controller
- Active Directory with users, OUs, and groups
- Working GPOs linked to your OUs
- A joined Windows client VM for testing

---

## Next Steps (Part 3 - Coming Soon)

- Configure File Shares with NTFS and Share permissions
- Enable Folder Redirection and Home Folders
- Add more realistic scenarios (e.g., locked account, password reset, GPO inheritance issues)
- Document ticket-style tasks for help desk lab simulation
