# Hyper-V-Lab-Deploying-a-Second-Domain-Controller-DC02-

## Purpose

This guide outlines the process of deploying `DC02` as a second Domain Controller and DNS server within your `mylabs.com` Active Directory domain. Adding a second DC provides critical redundancy, fault tolerance, and load balancing for your Active Directory and DNS services, making your lab environment more robust and mirroring a production setup.

## Prerequisites

Before starting this `DC02` deployment, ensure the following steps from the main lab build are completed and verified:

* **Hyper-V Host:** Hyper-V Manager is set up.
* **Virtual Switches:** `Internal_Lab_Switch` is created as an "Internal network."
* **DC01 Virtual Machine:**
    * `DC01` is fully configured as the first Domain Controller for `mylabs.com`.
    * DNS and DHCP roles are installed and running on `DC01`.
    * NAT is successfully configured on `DC01`, providing internet access to the `Internal_Lab_Switch` network.

## Step-by-Step Deployment and Configuration

These steps are performed on the Hyper-V Host and then within the `DC02` virtual machine.

### 1. Create DC02 Virtual Machine

1.  On your Hyper-V Host, open **Hyper-V Manager**.
2.  Click **"New"** > **"Virtual Machine..."**
3.  Follow the wizard:
    * **Name:** `DC02`
    * **Specify Generation:** Select **"Generation 2."**
    * **Assign Memory:** Enter `2048` MB (2 GB).
    * **Configure Networking:** For the "Connection" dropdown, select **`Internal_Lab_Switch`**. (`DC02` will *not* have an external adapter; it will rely on `DC01` for internet access).
    * **Connect Virtual Hard Disk:** Select "Create a virtual hard disk". Name it `DC02.vhdx`. Size: `60` GB.
    * **Installation Options:** Select "Install an operating system from a bootable image file" and browse to your Windows Server ISO file.
    * Click **Finish**.

### 2. Install Windows Server on DC02

1.  In Hyper-V Manager, right-click `DC02` and select **"Connect..."**.
2.  Click **"Start"** in the VM window.
3.  Follow the Windows Server installation prompts (language, time, currency, keyboard, then "Install now").
4.  Select your desired Windows Server edition (e.g., "Standard Evaluation (Desktop Experience)").
5.  Accept license terms.
6.  Select **"Custom: Install Windows only (advanced)"**.
7.  Select the `60 GB` drive for installation.
8.  Complete the installation. It will reboot multiple times.
9.  Set the local Administrator password when prompted after the final reboot.

### 3. Initial Configuration of DC02

1.  Log in to `DC02` as the local Administrator.
2.  **Rename the Computer:**
    * Right-click **Start** > **System** (or type `sysdm.cpl` in Run).
    * Click **"Change settings"** next to "Computer name".
    * Click **"Change..."**
    * In "Computer name:", type `DC02`.
    * Click OK, then OK again. **Restart `DC02`** when prompted.
3.  **Configure Static IP Address (Internal Adapter):**
    * After `DC02` restarts, log in as local Administrator.
    * Right-click **Start** > **Network Connections** > **"Change adapter options"** (or type `ncpa.cpl` in Run).
    * Identify the network adapter connected to `Internal_Lab_Switch`.
    * Right-click it > **Properties**.
    * Select **"Internet Protocol Version 4 (TCP/IPv4)"** > **Properties**.
    * Select "Use the following IP address:"
        * **IP address:** `192.168.10.2`
        * **Subnet mask:** `255.255.255.0`
        * **Default gateway:** `192.168.10.1` (Point to `DC01` as the gateway for internet access).
        * **Preferred DNS server:** `192.168.10.1` (Point to `DC01` for DNS resolution).
        * **Alternate DNS server:** `127.0.0.1` (Point to itself for DNS redundancy on DC02).
    * Click **OK**, then **Close**.
4.  **Perform Windows Updates:**
    * Go to **Settings** > **Windows Update**.
    * Click "Check for updates" and install all available updates. Restart `DC02` as needed.
    * Repeat until no more updates are found.
5.  **Verify Connectivity to DC01 and Internet:**
    * Open Command Prompt (Admin) on `DC02`.
    * `ping 192.168.10.1` (to `DC01`'s IP)
    * `ping DC01.mylabs.com` (to `DC01`'s name, testing DNS resolution via `DC01`)
    * `ping 8.8.8.8` (to external IP, testing internet access via `DC01`'s NAT)
    * `ping google.com` (to external domain, testing internet DNS resolution)
    * All pings should be successful.

### 4. Install Active Directory Domain Services (AD DS) and Promote DC02

This step adds `DC02` as a replica Domain Controller to your existing `mylabs.com` domain.

1.  On `DC02`, open **Server Manager**.
2.  Click **"Manage"** > **"Add Roles and Features."**
3.  Follow the wizard:
    * "Server Roles": Check **"Active Directory Domain Services."**
        * Click **"Add Features"** if prompted.
    * Click **Next** through Features and AD DS introductory pages.
    * "Confirmation": Click **"Install."**
4.  **Promote the Server to a Domain Controller:**
    * After the role installation completes, click the yellow exclamation mark in Server Manager (top right).
    * Click **"Promote this server to a domain controller."**
    * **Deployment Configuration:** Select **"Add a domain controller to an existing domain."**
    * **Domain:** Type `mylabs.com`.
    * Click **"Select..."** and provide `mylabs\Administrator` credentials if prompted. Click **OK**, then **Next**.
    * **Domain Controller Options:**
        * **Domain Name System (DNS) server:** Should be checked.
        * **Global Catalog (GC):** Should be checked.
        * **Read only domain controller (RODC):** Unchecked.
        * **Site name:** Select `Default-First-Site-Name`.
        * **Directory Services Restore Mode (DSRM) password:** Set a strong password (at least 8 characters, with uppercase, lowercase, numbers, and symbols). Ensure the "Next" button becomes active.
    * Click **Next** through DNS Options, Additional Options, Paths, and Review Options.
    * **Prerequisites Check:** Ensure all checks pass. Click **"Install."**
5.  The server will automatically **restart** after promotion.

## Verification

After `DC02` reboots and you log in as `mylabs\Administrator`, perform these checks.

1.  **Domain Membership:**
    * Right-click **Start** > **System**. Verify "Domain:" now shows `mylabs.com`.
2.  **Server Manager Roles:**
    * Open **Server Manager**. You should see "AD DS" and "DNS" roles listed with green status.
3.  **DNS Manager:**
    * Open **DNS Manager** (Tools > DNS). Verify `mylabs.com` forward and reverse lookup zones exist and are populated with records (especially for `DC01` and `DC02`).
4.  **Active Directory Replication:**
    * Open **Command Prompt (Admin)** on `DC02`.
    * Run: `repadmin /showrepl`
        * Look for "Last successful sync" messages for all listed Naming Contexts (Configuration, Schema, DomainDnsZones, ForestDnsZones, DC=mylabs,DC=com) between `DC02` and `DC01`. All should show successful and recent syncs.
    * Run: `dcdiag /test:dns`
        * Look for "Passed" for all tests related to DNS.
    * Verify SYSVOL and NETLOGON shares: `net share`
        * You should see `NETLOGON` and `SYSVOL` shares listed. This confirms Group Policy and logon scripts are replicating.
5.  **Test Object Replication (Creation/Deletion):**
    * On `DC02`, open **Active Directory Users and Computers (`dsa.msc`)**. Create a test user or group (e.g., "TestUserDC02").
    * Log in to `DC01` and open `dsa.msc`. Refresh the view. The "TestUserDC02" should appear. (This confirms replication from `DC02` to `DC01`).
    * On `DC01`, create another test user or group (e.g., "TestUserDC01").
    * Log in to `DC02` and open `dsa.msc`. Refresh the view. The "TestUserDC01" should appear. (This confirms replication from `DC01` to `DC02`).
    * Delete both test users after verification.

## Troubleshooting Tips

* **DSRM Password "Next" button disabled:** This almost always means the password does not meet the complexity requirements (min. 8 characters, 3 of 4 categories: uppercase, lowercase, numbers, symbols).
* **Replication Failures (`repadmin /showrepl` errors):**
    * **DNS:** This is the most common cause. Double-check DNS settings on *both* `DC01` and `DC02`. `DC01` should point to itself as primary, `DC02` as secondary. `DC02` should point to `DC01` as primary, itself as secondary.
    * **Firewall:** Ensure Windows Firewall isn't blocking necessary AD ports (though it's usually automatic for domain controllers).
    * **Time Skew:** Significant time differences (more than 5 minutes) between DCs can break replication. Verify time synchronization.
    * **`dcdiag /test:dns` errors:** Address any DNS-related issues reported by `dcdiag` first.
