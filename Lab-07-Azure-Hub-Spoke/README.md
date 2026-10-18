
# Lab 07 - Azure Hub-and-Spoke Network Architecture

## Overview

In this lab, I designed and implemented a basic Hub-and-Spoke network architecture in Microsoft Azure.

The main objective was to understand how multiple Azure Virtual Networks can be connected through a central Hub VNet using VNet Peering.

I also tested communication between two Spoke VNets and used Azure Network Watcher to investigate why connectivity failed.

The lab demonstrated an important Azure networking concept:

**VNet Peering is non-transitive by default.**

This means that two Spoke VNets connected to the same Hub VNet cannot automatically communicate with each other through the Hub.

---

## Lab Objectives

- Understand Hub-and-Spoke network architecture.
- Create a central Hub Virtual Network.
- Create two independent Spoke Virtual Networks.
- Configure VNet Peering between the Hub and each Spoke.
- Understand non-transitive routing.
- Deploy Linux virtual machines for connectivity testing.
- Test private communication between Spoke VNets.
- Troubleshoot routing using Azure Network Watcher.
- Understand how User Defined Routes (UDRs) and Network Virtual Appliances (NVAs) can enable communication between Spokes.
- Practice Azure resource cost management.

---

## Architecture

The lab environment consisted of three Azure Virtual Networks.

```text
                   Microsoft Azure
                         |
                    RG-LAB-07
                         |
                    HUB VNet
                   10.0.0.0/16
                         |
              +----------+----------+
              |                     |
         VNet Peering          VNet Peering
              |                     |
              |                     |
         Spoke 01               Spoke 02
       10.10.0.0/16           10.20.0.0/16
              |                     |
           snet-app               snet-db
        10.10.1.0/24           10.20.1.0/24
              |                     |
         vm-spoke01             vm-spoke02
         10.10.1.4             10.20.1.4
```

Both Spoke VNets were connected to the Hub through VNet Peering.

However, there was no direct peering connection between Spoke 01 and Spoke 02.

---

## 1. Resource Group

Created a dedicated Resource Group for the lab.

| Setting | Value |
|---|---|
| Resource Group | RG-LAB-07 |
| Region | East US |
| Subscription | Azure subscription 1 |

All lab resources were deployed within this Resource Group to simplify management and cleanup.

---

## 2. Hub Virtual Network

Created the central Virtual Network.

| Setting | Value |
|---|---|
| VNet Name | vnet-hub |
| Address Space | 10.0.0.0/16 |
| Subnet | snet-hub |
| Subnet Range | 10.0.1.0/24 |
| Region | East US |

### Purpose

The Hub VNet acts as the central network in the architecture.

In production environments, the Hub commonly hosts shared networking services such as:

- Azure Firewall
- Network Virtual Appliances (NVAs)
- VPN Gateway
- ExpressRoute Gateway
- Azure Bastion
- DNS services

In this lab, the Hub was used to demonstrate centralized network connectivity through VNet Peering.

No Azure Firewall or VPN Gateway was deployed to minimize costs.

---

## 3. Spoke 01 Virtual Network

Created the first Spoke VNet.

| Setting | Value |
|---|---|
| VNet Name | vnet-spoke-01 |
| Address Space | 10.10.0.0/16 |
| Subnet | snet-app |
| Subnet Range | 10.10.1.0/24 |
| Region | East US |

### Purpose

Spoke 01 represents an application network.

In a production environment, this network could host:

- Application servers
- Web servers
- Application services
- Internal workloads

The Spoke was deployed separately from the Hub to provide network isolation and independent management.

---

## 4. Spoke 02 Virtual Network

Created the second Spoke VNet.

| Setting | Value |
|---|---|
| VNet Name | vnet-spoke-02 |
| Address Space | 10.20.0.0/16 |
| Subnet | snet-db |
| Subnet Range | 10.20.1.0/24 |
| Region | East US |

### Purpose

Spoke 02 represents a backend or database network.

In a production environment, this network could host:

- Database servers
- Backend services
- Internal application databases
- Protected workloads

The three VNets were configured with non-overlapping address spaces.

This is required for standard Azure VNet Peering.

---

## 5. Address Space Overlap Troubleshooting

During the configuration, Azure displayed an address space overlap error.

The Hub and Spoke 01 initially had overlapping IP address ranges.

```text
vnet-hub       10.0.0.0/16
vnet-spoke-01  10.0.0.0/16
```

Azure rejected the peering configuration because the address spaces overlapped.

### Resolution

The Spoke 01 address space was corrected to:

```text
10.10.0.0/16
```

Final configuration:

```text
Hub VNet:    10.0.0.0/16
Spoke 01:    10.10.0.0/16
Spoke 02:    10.20.0.0/16
```

### Lesson Learned

Azure VNet Peering requires non-overlapping address spaces.

IP address planning is essential when designing cloud networks.

---

## 6. Configure Hub-to-Spoke Peering

Configured VNet Peering between the Hub and both Spoke VNets.

### Hub and Spoke 01

Peering connections:

```text
hub-to-spoke01
spoke01-to-hub
```

Configuration:

| Setting | Value |
|---|---|
| Virtual Network Access | Enabled |
| Forwarded Traffic | Disabled |
| Gateway Transit | Disabled |
| Use Remote Gateway | Disabled |

### Hub and Spoke 02

Peering connections:

```text
hub-to-spoke02
spoke02-to-hub
```

Configuration:

| Setting | Value |
|---|---|
| Virtual Network Access | Enabled |
| Forwarded Traffic | Disabled |
| Gateway Transit | Disabled |
| Use Remote Gateway | Disabled |

The peering connections were configured to allow communication between the Hub and each Spoke.

### Network Design

```text
Spoke 01 <------> Hub <------> Spoke 02
             VNet Peering
```

There was no direct peering connection between the two Spokes.

---

## 7. Understanding Peering Settings

### Allow Virtual Network Access

Allows resources inside peered VNets to communicate using private IP addresses.

This option was enabled for both Hub-to-Spoke connections.

### Allow Forwarded Traffic

Allows traffic forwarded by a device, such as a firewall or NVA, to cross a peering connection.

This option can be required when implementing centralized traffic inspection or routing.

It was disabled in this lab because no NVA was deployed.

### Gateway Transit

Allows a VNet to make its VPN Gateway or Route Server available to a peered VNet.

This is commonly used in centralized hybrid network architectures.

### Use Remote Gateway

Allows a VNet to use a gateway or route server located in another peered VNet.

Neither gateway transit nor remote gateway functionality was required in this lab.

---

## 8. Deploy Virtual Machines

Two Ubuntu Linux virtual machines were deployed to test network connectivity.

### VM - Spoke 01

| Setting | Value |
|---|---|
| VM Name | vm-spoke01 |
| Operating System | Ubuntu Server 24.04 LTS |
| VM Size | Standard_D2als_v7 |
| Virtual Network | vnet-spoke-01 |
| Subnet | snet-app |
| Private IP | 10.10.1.4 |

### VM - Spoke 02

| Setting | Value |
|---|---|
| VM Name | vm-spoke02 |
| Operating System | Ubuntu Server 24.04 LTS |
| Virtual Network | vnet-spoke-02 |
| Subnet | snet-db |
| Private IP | 10.20.1.4 |

Azure Run Command was used to execute connectivity tests.

This avoided the need to establish an SSH session from a local computer.

---

## 9. Connectivity Test Between Spokes

The objective was to test whether VM-Spoke01 could communicate with VM-Spoke02 through the Hub.

### Source

```text
VM: vm-spoke01
Private IP: 10.10.1.4
VNet: vnet-spoke-01
```

### Destination

```text
VM: vm-spoke02
Private IP: 10.20.1.4
VNet: vnet-spoke-02
```

### Test Command

Executed the following command using Azure Run Command on `vm-spoke01`:

```bash
ping -c 4 10.20.1.4
```

### Actual Result

```text
PING 10.20.1.4 (10.20.1.4) 56(84) bytes of data.

--- 10.20.1.4 ping statistics ---
4 packets transmitted, 0 received, 100% packet loss
```

### Observation

The connectivity test failed.

Although both Spoke VNets were connected to the Hub, there was no automatic transit routing between them.

The result was consistent with Azure VNet Peering's non-transitive behavior.

Further investigation was performed using Azure Network Watcher.

---

## 10. Troubleshooting with Azure Network Watcher

Used Azure Network Watcher to investigate the routing behavior.

Navigation:

```text
Azure Portal
    |
Network Watcher
    |
Network diagnostic tools
    |
Next hop
```

### Next Hop Configuration

| Setting | Value |
|---|---|
| Subscription | Azure subscription 1 |
| Resource Group | RG-LAB-07 |
| Virtual Machine | vm-spoke01 |
| Network Interface | vm-spoke01888 |
| Source IP | 10.10.1.4 |
| Destination IP | 10.20.1.4 |

### Actual Result

```text
Next hop type: None

IP address: -

Route table ID: System Route
```

### Analysis

The result showed that Azure did not have a usable next hop for traffic from Spoke 01 to Spoke 02.

The selected system route did not provide a valid forwarding path to the destination.

This confirmed that the configured Hub-to-Spoke peerings were not sufficient to provide transit connectivity between the Spokes.

---

## 11. Understanding Non-Transitive VNet Peering

Azure VNet Peering is non-transitive by default.

This means that if:

```text
VNet A <----> VNet B
VNet B <----> VNet C
```

It does not automatically mean:

```text
VNet A <----> VNet C
```

### Example from This Lab

```text
Spoke 01 <----> Hub
                  |
                  |
               Spoke 02
```

The Hub was connected to both Spokes.

However, the Hub did not automatically forward traffic between them.

### Important Concept

A VNet does not automatically become a router simply because it has peering connections to multiple VNets.

Additional routing components are required to enable transit connectivity through the Hub.

---

## 12. How to Enable Spoke-to-Spoke Communication

There are different ways to enable communication between Spoke VNets.

### Option 1 - Direct VNet Peering

Create a direct peering connection between Spoke 01 and Spoke 02.

```text
Spoke 01 <--------> Spoke 02
           Peering
```

This allows direct private communication without requiring traffic to pass through the Hub.

However, it does not provide centralized traffic inspection through the Hub.

### Option 2 - Hub with NVA or Azure Firewall

Deploy a routing or firewall appliance in the Hub.

```text
       Spoke 01
           |
           |
          UDR
           |
           v
      Hub Firewall
         / NVA
           |
           |
          UDR
           |
           v
       Spoke 02
```

This architecture can support centralized traffic inspection and routing.

Typical requirements include:

1. Deploy an Azure Firewall or NVA in the Hub.
2. Configure User Defined Routes (UDRs) on the Spoke subnets.
3. Configure the required forwarded traffic settings on VNet Peerings.
4. Enable IP forwarding when using a supported NVA.
5. Configure firewall rules and return routing.
6. Test connectivity between the Spokes.

These additional routing components were not implemented in this lab.

---

## 13. Connection to Previous Labs

This lab builds on concepts learned in earlier Azure networking labs.

### Lab 05 - Azure Routing, UDR and NVA

Learned how to:

- Create Azure Route Tables.
- Configure User Defined Routes.
- Configure an NVA.
- Enable IP forwarding.
- Control traffic paths using next-hop addresses.

### Lab 06 - Azure VNet Peering

Learned how to:

- Connect two Azure VNets.
- Configure local and remote peering.
- Enable private connectivity.
- Test communication between peered VNets.

### Lab 07 - Hub-and-Spoke

Combined the previous networking concepts to understand:

- Centralized Azure network architecture.
- Hub and Spoke network design.
- Multiple VNet Peering connections.
- Non-transitive routing.
- Routing limitations.
- Network troubleshooting.

---

## 14. Real-World Use Case

A company may use Hub-and-Spoke architecture to organize different workloads.

Example:

```text
                    On-Premises
                         |
                    VPN Gateway
                         |
                         |
                    Hub VNet
                         |
                   Azure Firewall
                         |
             +-----------+-----------+
             |                       |
             |                       |
       Application VNet         Database VNet
             |                       |
        Web Servers             SQL Servers
             |                       |
             +-----------------------+
```

### Benefits

- Centralized network management.
- Separation of workloads.
- Improved security architecture.
- Centralized firewall inspection.
- Easier hybrid connectivity.
- Scalable network design.

In a production environment, the exact routing and security configuration depends on the organization's requirements.

---

## 15. Cost Management

Cost optimization was an important part of this lab.

The following approach was used:

- Created a dedicated Resource Group.
- Avoided deploying Azure Firewall.
- Avoided deploying VPN Gateway.
- Used only two virtual machines.
- Used Azure Run Command for testing.
- Deallocated a VM while it was not required.
- Avoided unnecessary additional infrastructure.

### Cost Considerations

Azure Virtual Networks and subnets do not have a standalone hourly charge.

However, other resources may generate charges, including:

- Virtual machine compute.
- Managed disks.
- Public IP addresses.
- VNet Peering data transfer.

After completing the lab, the intended cleanup was to delete the dedicated Resource Group.

```text
RG-LAB-07
```

Deleting the Resource Group removes its contained resources after the deletion operation completes.

---

## 16. What I Learned

During this lab, I learned how to:

- Design an Azure Hub-and-Spoke architecture.
- Create a central Hub VNet.
- Deploy multiple Spoke VNets.
- Plan non-overlapping IP address spaces.
- Troubleshoot VNet address space conflicts.
- Configure Hub-to-Spoke VNet Peering.
- Understand local and remote peering settings.
- Understand non-transitive routing.
- Deploy Linux VMs for network testing.
- Test connectivity using private IP addresses.
- Use Azure Run Command.
- Troubleshoot routing with Azure Network Watcher.
- Interpret the Next Hop result.
- Understand the role of UDRs, NVAs and Azure Firewall.
- Apply basic cloud cost management practices.

---

## 17. Lab Results

| Test | Result |
|---|---|
| Hub VNet creation | Successful |
| Spoke 01 creation | Successful |
| Spoke 02 creation | Successful |
| Address space conflict resolution | Successful |
| Hub-to-Spoke peering configuration | Successful |
| VM deployment | Successful |
| Spoke-to-Spoke ping | Failed as expected |
| Network Watcher Next Hop | None |
| Non-transitive routing validation | Successful |

### Final Result

Successfully implemented a basic Azure Hub-and-Spoke network architecture.

The lab demonstrated that VNet Peering alone does not provide automatic transit routing between Spoke VNets.

Azure Network Watcher confirmed that traffic from Spoke 01 to Spoke 02 did not have a valid forwarding path.

**Lab 07 completed successfully.**