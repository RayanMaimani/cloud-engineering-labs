
# Lab 05 – Azure Routing, UDR & Network Virtual Appliance (NVA)

## Overview

In this lab, I implemented custom routing in Microsoft Azure using a User Defined Route (UDR) and a Linux Virtual Machine acting as a Network Virtual Appliance (NVA).

The goal was to understand how Azure routes traffic between subnets by default, how a Route Table can override Azure system routes, and how traffic can be forced through a specific network appliance such as a firewall or router.

---

## Lab Objectives

- Create an Azure Virtual Network with multiple subnets.
- Understand Azure system routing.
- Create an Azure Route Table.
- Create a User Defined Route (UDR).
- Deploy a Linux VM as a Network Virtual Appliance (NVA).
- Enable IP forwarding on the Azure NIC.
- Enable IPv4 forwarding inside Linux.
- Force traffic through the NVA.
- Test connectivity between subnets.
- Verify the actual next hop using Azure Network Watcher.

---

## Architecture

```text
                    vnet-lab-05
                    10.20.0.0/16

       ┌───────────────────────────────────────┐
       │                                       │
       │  snet-app                             │
       │  10.20.1.0/24                         │
       │                                       │
       │  vm-app                               │
       │  10.20.1.4                            │
       │      │                                │
       │      │ rt-app                         │
       │      │ UDR                            │
       │      ▼                                │
       │                                       │
       │  snet-nva                             │
       │  10.20.3.0/24                         │
       │                                       │
       │  vm-nva                               │
       │  10.20.3.4                            │
       │  IP Forwarding Enabled                │
       │      │                                │
       │      ▼                                │
       │                                       │
       │  snet-backend                         │
       │  10.20.2.0/24                         │
       │                                       │
       │  vm-backend                           │
       │  10.20.2.4                            │
       │                                       │
       └───────────────────────────────────────┘
```

Traffic flow:

```text
vm-app
10.20.1.4
    |
    | Destination: 10.20.2.4
    v
Route Table
rt-app
    |
    | UDR
    | Next Hop Type: Virtual Appliance
    v
vm-nva
10.20.3.4
    |
    | IP Forwarding
    v
vm-backend
10.20.2.4
```

---

# 1. Resource Group

A dedicated Resource Group was used for the lab:

```text
RG-LAB-05
```

Using a dedicated Resource Group makes it easier to manage and delete all lab resources after testing.

---

# 2. Virtual Network

A Virtual Network was created:

```text
Name: vnet-lab-05
Address Space: 10.20.0.0/16
```

The VNet was divided into three subnets.

### Application Subnet

```text
Name: snet-app
Address Range: 10.20.1.0/24
```

This subnet contains the application VM.

### Backend Subnet

```text
Name: snet-backend
Address Range: 10.20.2.0/24
```

This subnet contains the backend VM.

### NVA Subnet

```text
Name: snet-nva
Address Range: 10.20.3.0/24
```

This subnet contains the Network Virtual Appliance.

---

# 3. Understanding Azure Default Routing

Azure automatically creates system routes for every subnet.

Normally, because the following subnets belong to the same VNet:

```text
10.20.1.0/24
10.20.2.0/24
10.20.3.0/24
```

Azure can route traffic directly between them.

For example:

```text
vm-app
   |
   | Azure System Route
   v
vm-backend
```

No custom router is normally required.

The purpose of this lab was to override this behavior for specific traffic.

---

# 4. Route Table

A custom Route Table was created:

```text
Name: rt-app
```

The Route Table was associated with:

```text
snet-app
```

This means custom routes inside `rt-app` affect traffic leaving the application subnet.

---

# 5. User Defined Route (UDR)

A custom route was added to `rt-app`.

```text
Route Name:
route-to-backend-via-nva

Destination:
10.20.2.0/24

Next Hop Type:
Virtual Appliance

Next Hop IP:
10.20.3.4
```

This tells Azure:

> When traffic from `snet-app` needs to reach `10.20.2.0/24`, send it to the NVA at `10.20.3.4` instead of using the normal direct VNet route.

The resulting path becomes:

```text
snet-app
    |
    | UDR
    v
vm-nva
    |
    v
snet-backend
```

---

# 6. Network Virtual Appliance (NVA)

A Linux VM was deployed to act as the router.

```text
VM Name: vm-nva
Subnet: snet-nva
Private IP: 10.20.3.4
Operating System: Ubuntu Server 24.04 LTS
Public IP: None
```

The VM acts as an intermediary device between the application and backend networks.

In production environments, an NVA could instead be:

- Azure Firewall
- FortiGate
- Palo Alto Networks Firewall
- Cisco virtual appliance
- Check Point firewall
- Other routing/security appliances

---

# 7. Azure NIC IP Forwarding

By default, an Azure VM is not intended to forward traffic addressed to another machine.

IP forwarding was therefore enabled on the network interface attached to `vm-nva`.

```text
vm-nva
   |
   └── Network Interface
          |
          └── IP Forwarding: Enabled
```

This allows Azure to deliver transit traffic to the NVA.

---

# 8. Linux IPv4 Forwarding

IP forwarding also needed to be enabled inside the Ubuntu operating system.

The following commands were executed:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/99-ip-forward.conf
```

Verification:

```bash
sysctl net.ipv4.ip_forward
```

Expected result:

```text
net.ipv4.ip_forward = 1
```

Two forwarding configurations were therefore required:

```text
Azure NIC IP Forwarding
          +
Linux IPv4 Forwarding
          =
VM can operate as an NVA/router
```

---

# 9. Application VM

The source VM was deployed with:

```text
Name: vm-app
Subnet: snet-app
Private IP: 10.20.1.4
Public IP: None
```

Because `snet-app` was associated with `rt-app`, traffic from this VM destined for the backend subnet is affected by the UDR.

---

# 10. Backend VM

The destination VM was deployed with:

```text
Name: vm-backend
Subnet: snet-backend
Private IP: 10.20.2.4
Public IP: None
```

This VM was used as the destination for the routing test.

---

# 11. Connectivity Test

From `vm-app`, connectivity to `vm-backend` was tested using:

```bash
ping -c 4 10.20.2.4
```

Result:

```text
4 packets transmitted
4 received
0% packet loss
```

This confirmed successful connectivity between the application VM and backend VM.

However, successful ping alone does not prove which route Azure selected.

For this reason, Azure Network Watcher was used.

---

# 12. Network Watcher – Next Hop Verification

Azure Network Watcher's **Next Hop** diagnostic tool was used.

Test configuration:

```text
Source VM:
vm-app

Source IP:
10.20.1.4

Destination IP:
10.20.2.4
```

Azure returned:

```text
Next Hop Type:
VirtualAppliance

Next Hop IP Address:
10.20.3.4
```

This confirmed that Azure was using the custom route.

The verified traffic path was:

```text
vm-app
10.20.1.4
    |
    | UDR
    v
vm-nva
10.20.3.4
    |
    | IP Forwarding
    v
vm-backend
10.20.2.4
```

---

# 13. Key Concept – Route Table vs UDR

A Route Table is the container that holds custom routing rules.

Example:

```text
rt-app
```

A User Defined Route (UDR) is an individual routing rule inside that Route Table.

Example:

```text
Destination: 10.20.2.0/24
Next Hop: Virtual Appliance
Next Hop IP: 10.20.3.4
```

Simple analogy:

```text
Route Table = List of driving instructions

UDR = One instruction inside the list
```

---

# 14. What I Learned

Through this lab, I learned:

- How Azure routes traffic between subnets automatically.
- How Azure system routes work.
- How to create and associate a Route Table.
- What a User Defined Route (UDR) is.
- How route tables can override Azure's default routing behavior.
- What a Network Virtual Appliance (NVA) is.
- How a Linux VM can operate as a basic router.
- Why IP forwarding must be enabled on both Azure and the operating system.
- How to use `Virtual Appliance` as a UDR next-hop type.
- How to force traffic through a specific network device.
- How to test connectivity using ICMP/ping.
- Why successful connectivity alone does not prove the selected route.
- How to use Azure Network Watcher Next Hop to verify routing decisions.

---

# 15. Real-World Use Case

The same concept is commonly used in enterprise Azure environments.

For example:

```text
Application Subnet
       |
       v
Route Table / UDR
       |
       v
FortiGate Firewall
       |
       v
Backend / Database Network
```

Instead of allowing workloads to communicate directly, organizations can force traffic through a firewall for:

- Security inspection
- Traffic filtering
- Logging
- Access control
- Network segmentation
- Threat prevention

The Linux NVA used in this lab represents the routing concept behind these enterprise designs.

---

# 16. Cost Management

This lab used temporary Azure resources.

The virtual machines were created only for testing and the dedicated Resource Group was deleted after completion:

```text
RG-LAB-05
```

Deleting the lab Resource Group removes the resources contained within it and prevents those deleted resources from continuing to generate usage charges.

---

## Lab Result

**Status: Successfully Completed**

Verified:

```text
vm-app (10.20.1.4)
        |
        | UDR
        v
vm-nva (10.20.3.4)
        |
        | IP Forwarding
        v
vm-backend (10.20.2.4)
```

Azure Network Watcher confirmed:

```text
Next Hop Type: VirtualAppliance
Next Hop IP:   10.20.3.4
```

The lab successfully demonstrated custom Azure routing using a Route Table, User Defined Route, and Network Virtual Appliance.