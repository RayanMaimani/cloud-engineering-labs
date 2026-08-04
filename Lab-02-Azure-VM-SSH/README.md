
# Lab 02 - Azure Virtual Machine & SSH

##  Goal

Deploy an Ubuntu Virtual Machine in Microsoft Azure and securely connect to it using SSH.

---

##  Concept

This lab focused on deploying a Linux Virtual Machine inside an Azure Virtual Network and securing remote access using SSH key authentication.

Network Security Groups (NSGs) were used to control inbound traffic.

---

##  Azure Resources

| Resource | Name |
|----------|---------------------|
| Virtual Machine | vm-managment-lab |
| Operating System | Ubuntu Server 24.04 LTS |
| Authentication | SSH Public Key |
| Username | cloudadmin |
| Public IP | Enabled |
| Network Security Group | NSG Attached |
| SSH Port | TCP 22 |

---

##  Security

The VM was initially deployed with **no public inbound ports**.

A temporary NSG rule was created to allow SSH access from the Internet.

After verifying connectivity, the environment can be secured again by removing the SSH rule.

---

##  SSH Connection

Successfully connected from Windows PowerShell using:

```bash
ssh -i vm-mangment-lab_key.pem cloudadmin@20.25.113.58
```

---

##  What I Learned

- Azure Virtual Machine deployment
- Ubuntu Server on Azure
- SSH key authentication
- Public IP concepts
- Network Security Groups (NSG)
- Temporary firewall rules
- Secure remote administration

---

## 🏢 Real World Usage

Most production environments do not leave SSH (TCP/22) open to the Internet.

Instead, organizations typically use:

- Azure Bastion
- Point-to-Site VPN
- Site-to-Site VPN
- Jump Servers
- Private IP addressing

Public SSH was used in this lab for learning purposes only.

---

## 🚀 Next Lab

Azure Linux Administration

- Linux filesystem
- Users & permissions
- Package management
- Networking commands
- File operations