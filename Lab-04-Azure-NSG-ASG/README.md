
# Lab 04 – Azure Network Security with NSG & ASG

## Overview

In this lab, I implemented Azure network security using **Network Security Groups (NSGs)** and **Application Security Groups (ASGs)**.

The goal was to control SSH traffic between an Admin VM and a Web VM using role-based security instead of relying only on IP addresses.

The lab also included real troubleshooting scenarios involving incorrect NSG association and incorrect ASG membership.

---

## Architecture

```text
                    Azure VNet
                   10.0.0.0/16
                         |
          --------------------------------
          |                              |
     snet-admin                     snet-server
     10.0.1.0/24                    10.0.2.0/24
          |                              |
      vm-admin                         vm-web
      10.0.1.4                         10.0.2.4
          |                              |
      asg-admin                        asg-web
          |                              |
          |-------- SSH TCP/22 -------->|
                         |
                    nsg-server
```

---

## Azure Resources

| Resource | Configuration |
|---|---|
| Resource Group | RG-LAB-04 |
| Virtual Network | vnet-lab-04 |
| Address Space | 10.0.0.0/16 |
| Admin Subnet | snet-admin – 10.0.1.0/24 |
| Server Subnet | snet-server – 10.0.2.0/24 |
| Admin VM | vm-admin – 10.0.1.4 |
| Web VM | vm-web – 10.0.2.4 |
| Admin ASG | asg-admin |
| Web ASG | asg-web |
| Server NSG | nsg-server |

---

## Application Security Groups

Two Application Security Groups were created.

### asg-admin

`vm-admin` was assigned to:

```text
asg-admin
```

This ASG represents systems authorized to perform administrative tasks.

### asg-web

`vm-web` was assigned to:

```text
asg-web
```

This ASG represents Web Servers.

Instead of creating security rules using individual IP addresses:

```text
10.0.1.4 → 10.0.2.4 → TCP/22
```

ASGs allow the security policy to use logical server roles:

```text
asg-admin → asg-web → TCP/22
```

This makes security policies easier to manage and scale.

---

## Network Security Group

A Network Security Group was created:

```text
nsg-server
```

The NSG was associated with:

```text
vnet-lab-04
    |
    └── snet-server
        10.0.2.0/24
```

The NSG controls traffic entering the Server subnet.

---

## NSG Rule 1 – Allow Admin SSH

The first custom inbound rule allows Admin systems to connect to Web Servers using SSH.

| Setting | Value |
|---|---|
| Name | Allow-Admin-to-Web-SSH |
| Priority | 100 |
| Source | asg-admin |
| Destination | asg-web |
| Protocol | TCP |
| Destination Port | 22 |
| Action | Allow |

Traffic:

```text
asg-admin
    |
    | SSH TCP/22
    v
asg-web

ALLOW
```

---

## NSG Rule 2 – Deny Other SSH Traffic

A second rule blocks SSH connections from other sources.

| Setting | Value |
|---|---|
| Name | Deny-All-to-Web-SSH |
| Priority | 200 |
| Source | Any |
| Destination | asg-web |
| Protocol | TCP |
| Destination Port | 22 |
| Action | Deny |

Traffic:

```text
Any
 |
 | SSH TCP/22
 v
asg-web

DENY
```

---

## Understanding NSG Priority

Azure evaluates NSG rules based on their priority.

**Lower numbers have higher priority.**

The security policy used in this lab was:

```text
Priority 100
asg-admin → asg-web → SSH → ALLOW

Priority 200
Any → asg-web → SSH → DENY
```

If `vm-admin` attempts to SSH to `vm-web`, the traffic matches Priority 100 first.

Azure allows the connection and stops processing additional rules.

If another system attempts SSH access, it does not match Priority 100 and continues to Priority 200, where the connection is denied.

---

## Default Azure NSG Rules

Azure automatically creates several default NSG rules, including:

```text
AllowVNetInBound
AllowAzureLoadBalancerInBound
DenyAllInBound
```

One important default rule is:

```text
Priority 65000
AllowVNetInBound
```

This normally allows communication between resources inside the same Virtual Network.

Our custom Priority 100 and 200 rules are processed before the Azure default rules.

---

## SSH Connectivity Test

I first connected from my local workstation to the Admin VM using SSH.

After connecting to `vm-admin`, I tested connectivity to the Web VM using its private IP address:

```bash
ssh azureuser@10.0.2.4
```

The connection succeeded.

The traffic path was:

```text
vm-admin
10.0.1.4
    |
    | Member of asg-admin
    |
    | SSH TCP/22
    v
nsg-server
    |
    | Priority 100
    | ALLOW
    v
asg-web
    |
    v
vm-web
10.0.2.4
```

---

## Deny Rule Validation

To validate the security policy, `vm-admin` was temporarily removed from `asg-admin`.

The expected traffic evaluation was:

```text
vm-admin
    |
    | Not a member of asg-admin
    v

Priority 100
No Match
    |
    v
Priority 200
DENY
```

After correcting the NSG configuration, SSH access was successfully blocked.

`vm-admin` was then added back to `asg-admin`.

SSH was tested again:

```bash
ssh azureuser@10.0.2.4
```

The connection succeeded.

This confirmed that ASG membership and NSG priority were controlling the traffic correctly.

---

# Troubleshooting

## Issue 1 – NSG Associated with the Wrong Subnet

During testing, SSH continued to work even when the Deny rule should have blocked the connection.

The NSG rules were checked and appeared correct.

The NSG association was then inspected.

The problem was discovered:

`nsg-server` was associated with an old subnet instead of the Server subnet used in this lab.

### Incorrect Association

```text
nsg-server
    |
    v
network-lab
    |
    v
snet-vm
```

Because the NSG was associated with the wrong subnet, its security rules were not being applied to `vm-web`.

### Fix

The incorrect association was removed.

The NSG was associated with the correct subnet:

```text
nsg-server
    |
    v
vnet-lab-04
    |
    v
snet-server
10.0.2.0/24
    |
    v
vm-web
10.0.2.4
```

After correcting the NSG association, the Deny rule worked successfully.

---

## Issue 2 – Incorrect ASG Membership

During further testing, SSH access was still blocked after the Admin VM was supposed to be returned to the Admin ASG.

The ASG membership was inspected.

The problem was:

```text
vm-admin → asg-web
```

instead of:

```text
vm-admin → asg-admin
```

Because `vm-admin` was not a member of `asg-admin`, the Priority 100 Allow rule did not match.

The traffic therefore reached the Priority 200 Deny rule.

### Fix

The ASG membership was corrected:

```text
vm-admin → asg-admin
vm-web   → asg-web
```

SSH was tested again:

```bash
ssh azureuser@10.0.2.4
```

The connection succeeded.

---

## Final Security Policy

The final configuration was:

```text
vm-admin
10.0.1.4
    |
    v
asg-admin
    |
    | SSH TCP/22
    v
+----------------------------+
|         nsg-server         |
|                            |
| Priority 100               |
| asg-admin → asg-web        |
| SSH → ALLOW                |
|                            |
| Priority 200               |
| Any → asg-web              |
| SSH → DENY                 |
+----------------------------+
    |
    v
asg-web
    |
    v
vm-web
10.0.2.4
```

---

## NSG vs ASG

A simple way to remember the difference:

```text
ASG = WHO is the resource?

NSG = WHAT traffic is allowed?
```

For example:

```text
ASG:
This VM belongs to the Admin group.

NSG:
Admin systems are allowed to SSH to Web Servers.
```

---

## Security Principle – Least Privilege

Instead of allowing:

```text
Any → Web Server → SSH
```

the configuration allows:

```text
asg-admin → asg-web → SSH
```

and denies other SSH sources.

This follows the **Principle of Least Privilege**, where systems receive only the network access they require.

---

## Validation Results

| Test | Result |
|---|---|
| Local Workstation → vm-admin SSH | PASS |
| vm-admin → vm-web SSH | PASS |
| vm-admin removed from asg-admin → vm-web SSH | BLOCKED |
| vm-admin added back to asg-admin → vm-web SSH | PASS |

---

## Troubleshooting Lessons

When an NSG rule does not behave as expected, verify:

1. Source and destination
2. Protocol and port
3. Rule priority
4. Allow/Deny action
5. NSG association
6. Correct subnet
7. ASG membership
8. VM private IP
9. Effective traffic path

A correctly configured NSG rule will not have any effect if the NSG is associated with the wrong subnet.

Similarly, an ASG-based rule will not match if the VM is assigned to the wrong ASG.

---

## What I Learned

During this lab I learned:

- How Azure Network Security Groups work
- How Application Security Groups work
- The difference between NSG and ASG
- How to create inbound NSG rules
- How NSG Priority works
- Why lower priority numbers are evaluated first
- How Azure default NSG rules work
- How to associate an NSG with a subnet
- How to assign VM Network Interfaces to ASGs
- How to allow SSH between specific application groups
- How to deny unauthorized SSH traffic
- How to implement Least Privilege in Azure networking
- How to test security rules using real SSH traffic
- How to troubleshoot incorrect NSG associations
- How to troubleshoot incorrect ASG memberships

---

## Final Result

The lab successfully implemented role-based network security using Azure NSGs and ASGs.

The Admin VM was allowed to access the Web VM using SSH while unauthorized SSH traffic was blocked.

The lab also provided practical troubleshooting experience by identifying and correcting both an incorrect NSG subnet association and an incorrect ASG membership.

**Lab Status: COMPLETED**