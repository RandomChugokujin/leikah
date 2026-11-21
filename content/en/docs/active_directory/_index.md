---
title: Active Directory
description: Enumerate and compromise networks running Microsoft Active Directory
categories: [Active Directory]
tags: [Active Directory]
weight: 2
---

Active Directory is Microsoft's directory service for Windows domain networks. It centralizes identity information and **authentication** through **Domain Controllers (DCs)**, which are Windows servers running **Active Directory Domain Services (AD DS)**. This information also supports **authorization**, which resource servers enforce when users access resources. Active Directory is very commonly used to manage internal enterprise networks.

The network protocols most essential for the function of **AD DS** are:
- **Kerberos**: A ticket-based authentication protocol that allows domain accounts to authenticate to resources and services; those services enforce authorization.
- **Lightweight Directory Access Protocol (LDAP)**: An internet directory access protocol used by Active Directory for the organization and retrieval of directory data on the domain.

Other services and protocols that are used within Active Directory environments include:
- **NTLM Authentication**: A legacy challenge-response authentication protocol that is still supported in many AD environments as a fallback option and can be abused in **Pass-the-Hash** attacks.
- **Active Directory Certificate Services (AD CS)**: Allows Windows servers to act as **Certificate Authorities (CAs)** and issue and manage **public key infrastructure (PKI)** certificates used for secure communication and authentication on the domain.
- **Other network services and protocols**: SMB, RDP, WinRM, MSSQL, HTTP, etc.


{{% alert title="Note" %}}
Linux hosts can also be part of an Active Directory domain when configured properly and can thus authenticate domain accounts and access domain services.
{{% /alert %}}

Active Directory services become a key way for attackers to gain initial access, lateral movement, privilege escalation, and eventually full domain compromise. Once attackers breach the domain initially, they can harvest hashes and credentials of domain accounts and abuse their access rights to move laterally within the network. This process is repeated until the attacker leverages their access to compromise a domain admin or enterprise admin user. From there, attackers can dump the password hashes stored on the domain controller, or use any of the plethora of methods to establish persistent and privileged access on the domain.
