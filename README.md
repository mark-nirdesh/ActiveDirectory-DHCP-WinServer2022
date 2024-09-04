

# How to Set Up a Basic Home Lab Running Active Directory (Oracle VirtualBox) and Add Users with PowerShell

## Overview
In this guide, I will walk you through the steps to create an Active Directory lab on your personal computer using Oracle VirtualBox. We will set up a domain controller, add users programmatically using PowerShell, and configure networking components to simulate a mini corporate network. This lab is ideal for learning how Active Directory works and enhancing your understanding of Windows networking.
## Note
I have done this in an extension to my home SOC lab, so I don't have a screenshot for Virtualbox because I am using Proxmox Type 1 Hypervisor, other than that I have used 10.10.10.0/24 network pool for my SOC lab but for this tutorial, I have used 172.16.0.0/24.

### Contents
- [Prerequisites](#prerequisites)
- [Step 1: Install Oracle VirtualBox](#step-1-install-oracle-virtualbox)
- [Step 2: Download Windows 10 and Server 2022 ISOs](#step-2-download-windows-10-and-server-2022-isos)
- [Step 3: Set Up the Domain Controller](#step-3-set-up-the-domain-controller)
- [Step 4: Network Configuration](#step-4-network-configuration)
- [Step 5: Install Active Directory](#step-5-install-active-directory)
- [Step 6: Set Up DHCP on the Domain Controller](#step-6-set-up-dhcp-on-the-domain-controller)
- [Step 7: Add Users with PowerShell](#step-7-add-users-with-powershell)
- [Step 8: Set Up a Windows 10 Client](#step-8-set-up-a-windows-10-client)
- [Conclusion](#conclusion)

---

### Prerequisites
Before we begin, ensure you have:
- Oracle VirtualBox installed
- Windows 10 and Server 2022 ISO images downloaded
- At least 8GB of RAM for VirtualBox to run smoothly
- Basic understanding of networking and virtual machines

---

### Step 1: Install Oracle VirtualBox
The first step is to download and install Oracle VirtualBox, which we will use to create and run our virtual machines.

1. Visit the [VirtualBox website](https://www.virtualbox.org/).
2. Download the appropriate version for your operating system (Windows, macOS, or Linux).
3. Install VirtualBox and, during the process, also download and install the VirtualBox Extension Pack, which enhances VirtualBox functionalities like USB device support.

---

### Step 2: Download Windows 10 and Server 2022 ISOs
We need two operating systems:
- Windows 10 (Client)
- Server 2022 (Domain Controller)

1. Go to the official Microsoft websites for [Windows 10](https://www.microsoft.com/software-download/windows10ISO) and [Server 2022](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2022) and download the ISO files.
2. Store the ISO files in an accessible location (e.g., your desktop).

---

### Step 3: Set Up the Domain Controller
In VirtualBox, we'll first create a virtual machine (VM) to serve as our Domain Controller (DC), running Windows Server 2022.

1. Open VirtualBox and click `New`.
2. Name the VM `DC` (Domain Controller) and select "Windows 2022 64-bit."
3. Assign **2GB of RAM** and accept all other defaults.
4. Before starting the VM, configure the network by adding two NICs:
   - One for external communication (NAT).
   - One for internal communication (Internal Network).

After creating the VM:
1. Boot the VM and attach the Server 2022 ISO.
2. Install Server 2022, and when prompted, select the "Desktop Experience" version.
3. Set the password for the administrator as `Password1`.

---

### Step 4: Network Configuration
After setting up the server, configure the network for internal and external communication:

1. Rename the two network adapters:
   - One as `Internet` (NAT).
   - One as `Internal` (Internal Network).
2. Assign the internal NIC a static IP of `172.16.0.1/24`, and set the DNS server to the same IP (i.e., the domain controller will act as its DNS server).

---

### Step 5: Install Active Directory
Now we will install Active Directory on the domain controller:

1. Open the **Server Manager** and click on `Add Roles and Features`.
2. Select **Active Directory Domain Services** and follow the wizard to install it.
3. Promote the server to a domain controller by creating a new forest named `mydomain.com`.
4. The server will reboot, and Active Directory will be installed.

---

### Step 6: Set Up DHCP on the Domain Controller

To ensure that your client machines receive dynamic IP addresses and network configuration automatically, we need to set up DHCP on the Domain Controller. This section explains how to install and configure DHCP to work with your internal virtual network.

#### 1. Install DHCP Server
   - Open **Server Manager**.
   - Navigate to **Add Roles and Features**.
   - In the list of available roles, select **DHCP Server** and follow the wizard to complete the installation.
   - After installation, click **Complete DHCP configuration** to finalize the setup.

#### 2. Create a DHCP Scope
   A DHCP scope defines the pool of IP addresses the server will assign to the client machines.
   - Go to **Tools** > **DHCP** from Server Manager.
   - Right-click on **IPv4** and select **New Scope**.
   - Name your scope (e.g., `Internal_Network_Scope`) and configure the IP address range:
     - **Start IP**: `172.16.0.100`
     - **End IP**: `172.16.0.200`
     - **Subnet Mask**: `255.255.255.0` (for a `/24` subnet)
   - Proceed through the wizard, leaving other options at their default unless specified below.

#### 3. Configure DHCP Options
   After creating the scope, configure the network options that will be assigned to clients, including the default gateway and DNS server.
   
   - Right-click on the scope you created and select **Configure Options**.
   - In the **Router** (default gateway) option, add:
     - **Default Gateway**: `172.16.0.1` (the internal IP of your Domain Controller).
   - In the **DNS Servers** option, add:
     - **DNS Server**: `172.16.0.1` (the Domain Controller, which is also the DNS server).

#### 4. Activate the Scope
   - Right-click on the new scope and select **Activate**. This will start the DHCP service and allow it to assign IP addresses to any client that requests one.

#### 5. Verify DHCP Leases
   Once the DHCP scope is activated, any new client machine connected to the internal network will automatically request an IP address from the DHCP server.
   - On a client machine (e.g., the Windows 10 client), open a command prompt and type:
     ```bash
     ipconfig /all
     ```
   - You should see an IP address assigned from the range you configured (e.g., `172.16.0.100`), along with the correct default gateway (`172.16.0.1`) and DNS server (`172.16.0.1`).

---

### Common Troubleshooting for DHCP:
- If the client is not receiving an IP address, verify the following:
  - The DHCP scope is activated.
  - The client machine is connected to the correct internal network.
  - DHCP options such as the default gateway and DNS server are properly configured.
  - Restart the DHCP service if needed by going to **DHCP** > Right-click on your server > **Restart Services**.

---

### Step 7: Add Users with PowerShell
To bulk create users in Active Directory, we will use a PowerShell script:

1. Download the script from the provided link (included in the project).
2. Disable Internet Explorer Enhanced Security to allow downloading the script directly on the domain controller.
3. Edit the script to include a list of usernames stored in a text file.
4. Open PowerShell as an Administrator and run the script:
   ```powershell
   Set-ExecutionPolicy Unrestricted
   .\Create-Users.ps1
   ```
   This script will create 1,000 users with default credentials (`Password1`) in Active Directory.

---

### Step 8: Set Up a Windows 10 Client
Next, we will create another virtual machine running Windows 10, which will act as a client:

1. In VirtualBox, create a new VM named `Client1` and select **Windows 10 64-bit**.
2. Set up the network adapter to use **Internal Network**, connecting it to the internal network for the lab.
3. Boot the VM using the Windows 10 ISO, and install Windows 10.
4. After installation, join the Windows 10 client to the domain (`mydomain.com`) using one of the users created earlier.

---

### Conclusion
You now have a fully functional Active Directory lab environment running on VirtualBox. This lab includes a domain controller with users created via PowerShell and a Windows 10 client that is part of the domain. This setup simulates a small-scale corporate network and is a fantastic way to practice AD and networking skills.

Feel free to clone this repository and use the provided resources and PowerShell scripts to replicate this lab. If you encounter any issues, please contact me!

---
