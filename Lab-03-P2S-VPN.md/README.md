
# Azure Lab 03 – Point-to-Site VPN (P2S)

## Objective

Create a secure Point-to-Site VPN connection between a Windows client and an Azure Virtual Network, then access a Linux VM using its private IP address via SSH.

---

## Architecture

Windows Laptop
      |
      | IKEv2 / Certificate Authentication
      |
   Internet
      |
Azure VPN Gateway
      |
Azure Virtual Network
10.0.0.0/16
      |
VM Subnet
10.0.1.0/24
      |
Linux VM
10.0.1.4

---

## Azure Resources

Resource Group:
- RG-LAB-02

Virtual Network:
- network-lab-2
- Address Space: 10.0.0.0/16

Subnets:
- VM Subnet: 10.0.1.0/24
- GatewaySubnet

Virtual Machine:
- vm-linux-lab
- OS: Ubuntu Linux
- Private IP: 10.0.1.4
- Username: azureuser

VPN Gateway:
- vpn-lab
- Gateway Type: VPN
- SKU: Basic

Point-to-Site Configuration:
- Client Address Pool: 172.16.0.0/24
- Tunnel Type: IKEv2
- Authentication: Azure Certificate

---

## Certificate Authentication

Two certificates were created using PowerShell.

### Root Certificate

P2SRootCert

The public portion of this certificate was uploaded to the Azure VPN Gateway.

### Client Certificate

P2SClientCert

The client certificate was signed using the P2SRootCert and installed in the Windows Current User certificate store.

Certificate authentication allows Azure VPN Gateway to authenticate the VPN client without using usernames and passwords.

---

## VPN Client

The VPN client configuration was downloaded from Azure VPN Gateway.

For Windows 64-bit:

WindowsAmd64/VpnClientSetupAmd64.exe

After installation, Windows created the VPN connection:

network-lab-2

The VPN successfully connected to Azure.

---

## Connectivity Test

After establishing the VPN connection, SSH access to the Linux VM was tested using its PRIVATE IP address.

Command:

ssh azureuser@10.0.1.4

Result:

azureuser@vm-linux-lab:~$

SSH connection was successful.

The VM also confirmed:

IPv4 address for eth0: 10.0.1.4

---

## Traffic Flow

Laptop
   ↓
Point-to-Site VPN
   ↓
Azure VPN Gateway
   ↓
Azure VNet
   ↓
VM Subnet
   ↓
Linux VM (10.0.1.4)
   ↓
SSH TCP/22

The VM was accessed through its private Azure network instead of directly exposing SSH access to the Internet.

---

## What I Learned

- How Azure Point-to-Site VPN works.
- The purpose of Azure VPN Gateway.
- The purpose of GatewaySubnet.
- How VPN clients receive addresses from a separate VPN address pool.
- How certificate-based VPN authentication works.
- Difference between a Root Certificate and Client Certificate.
- How to configure IKEv2 P2S connectivity.
- How to install an Azure VPN client profile on Windows.
- How a local computer can communicate with Azure private IP addresses.
- How to access a Linux VM securely using SSH over a VPN.
- Why private connectivity is preferable to exposing management ports directly to the Internet.

---

## Result

✅ Point-to-Site VPN connected successfully.

✅ Windows client received secure connectivity to the Azure VNet.

✅ Linux VM was reachable through its private IP address.

✅ SSH connection to 10.0.1.4 succeeded.

## Lab Status

COMPLETED ✅