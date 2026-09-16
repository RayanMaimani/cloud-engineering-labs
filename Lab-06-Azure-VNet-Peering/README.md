
# Lab 06 - Azure VNet Peering

## Overview

In this lab, I configured Azure VNet Peering to connect two separate Azure Virtual Networks.

The goal was to allow virtual machines in different VNets to communicate using their private IP addresses without using public IP addresses or a VPN Gateway.

---

## Architecture

```text
vm-a
10.10.1.4
   |
snet-a
10.10.1.0/24
   |
vnet-a
10.10.0.0/16
   |
   |  VNet Peering
   |
vnet-b
10.20.0.0/16
   |
snet-b
10.20.1.0/24
   |
vm-b
10.20.1.4
```

---

## Resources

### Resource Group

```text
RG-LAB-06
```

Region:

```text
East US
```

### Virtual Network A

```text
Name: vnet-a
Address Space: 10.10.0.0/16
Subnet: snet-a
Subnet Range: 10.10.1.0/24
```

### Virtual Network B

```text
Name: vnet-b
Address Space: 10.20.0.0/16
Subnet: snet-b
Subnet Range: 10.20.1.0/24
```

The VNets were configured with non-overlapping address spaces, which is required for VNet Peering.

---

## Virtual Machines

Two Ubuntu virtual machines were deployed.

### VM-A

```text
Name: vm-a
VNet: vnet-a
Subnet: snet-a
Private IP: 10.10.1.4
Public IP: None
```

### VM-B

```text
Name: vm-b
VNet: vnet-b
Subnet: snet-b
Private IP: 10.20.1.4
Public IP: None
```

Public IP addresses were not required because all connectivity testing was performed using private IP addresses.

---

## Test Before VNet Peering

Before configuring VNet Peering, I tested connectivity from `vm-a` to `vm-b`.

From `vm-a`:

```bash
ping -c 4 10.20.1.4
```

Result:

```text
100% packet loss
```

The test failed because `vnet-a` and `vnet-b` were separate virtual networks with no connectivity between them.

---

## Configure VNet Peering

VNet Peering was configured between:

```text
vnet-a <------> vnet-b
```

Network access was enabled in both directions:

```text
vnet-a -> vnet-b : Allowed
vnet-b -> vnet-a : Allowed
```

The following advanced options were not required:

```text
Forwarded Traffic: Disabled
Gateway Transit: Disabled
Use Remote Gateway: Disabled
```

---

## Peering Settings

### Allow Virtual Network Access

Allows resources inside the two VNets to communicate through the peering connection.

This option was enabled.

### Forwarded Traffic

Allows traffic forwarded by another device, such as a firewall or Network Virtual Appliance (NVA), to cross the peering connection.

This option was not required for this lab.

### Gateway Transit

Allows a peered VNet to use a VPN Gateway or Route Server through another VNet.

This option was not required.

### Use Remote Gateway

Allows a VNet to use a gateway located in the remote peered VNet.

This option was not required.

---

## Test After VNet Peering

After the peering status became `Connected`, I repeated the same connectivity test from `vm-a`.

```bash
ping -c 4 10.20.1.4
```

Result:

```text
Connection successful
0% packet loss
```

The same private IP that was unreachable before the peering became reachable after VNet Peering was configured.

---

## Before vs After

### Before Peering

```text
vm-a -> vnet-a     X     vnet-b -> vm-b

100% packet loss
```

### After Peering

```text
vm-a -> vnet-a <------> vnet-b -> vm-b
                Peering

0% packet loss
```

---

## Key Concepts Learned

- A VNet is an isolated private network in Azure.
- Separate VNets do not automatically communicate with each other.
- VNet Peering provides private connectivity between Azure VNets.
- Peered VNets must have non-overlapping IP address spaces.
- Resources can communicate through peering using private IP addresses.
- Public IP addresses are not required for communication across VNet Peering.
- Peering access can be configured for both directions.
- Forwarded Traffic is useful when firewalls or NVAs forward traffic between networks.
- Gateway Transit can be used when VNets need to share gateway connectivity.

---

## Real-World Example

An organization could separate workloads into different VNets:

```text
Application VNet
       |
       | VNet Peering
       |
Database VNet
```

Another common design is:

```text
Spoke VNet
    |
    | Peering
    |
Hub VNet
    |
Firewall / VPN Gateway
```

This type of architecture is commonly used when building larger Azure network environments.

---

## What I Learned

In this lab, I learned how to:

- Create multiple Azure VNets
- Design non-overlapping IP address spaces
- Deploy VMs into separate VNets
- Test connectivity before making network changes
- Configure Azure VNet Peering
- Understand local and remote peering settings
- Understand forwarded traffic and gateway options
- Test private connectivity after peering

---

## Lab Result

Azure VNet Peering was successfully configured between `vnet-a` and `vnet-b`.

Before peering, communication between the two VMs failed.

After peering, `vm-a` successfully communicated with `vm-b` using its private IP address.

**Lab completed successfully.**