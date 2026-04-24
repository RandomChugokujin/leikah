---
title: Active Directory
description: Enumerate and compromise networks running Microsoft Active Directory
categories: [Active Directory]
tags: [Active Direcotory]
weight: 2
---

Active Directory is Microsoft's directory service for Windows domain networks. Its primary goal is to centralize the **authentication** and **authorization** of users within the network to **Domain Controllers (DC)**, which are commonly Windows servers running the **Active Directory Domain Service (AD DS)**. Active Directory is very commonly used to manage internal enterprise networks.

The network protocols most essential for the function of **AD DS** are:
- **Kerberos**: A ticket-based authentication protocol that allows the DCs to authenticate user logins and authorize users' access to resources and services.
- **Lightweight Directory Access Protocol (LDAP)**: An internet directory access protocol used by Active Directory for the organization and retreival of directory data on the domain.

Other Services and protocols that are used within Active Directory environments include:
- **NTLM Authentication**: A legacy challenge-response authentication protocol that is still supported by AD as a fallback option and vulnerable to **Pass-the-Hash** attack.
- **Active Directory Certificate Service (AD CS)**: Allows domain servers to act as **Certificate Authorities (CA)**, issue and manage **public key infrastructure (PKI)** certificates used for secure communication and authentication on the domain.
- **Other Network protocols**: SMB, RDP, WinRM, MSSQL, HTTP, etc.


{{% alert title="Note" %}}
Linux hosts can also be part of an Active Directory domain when configured properly and can thus authenticate domain accounts and access domain services.
{{% /alert %}}

Active Directory services becomes a key way for attackers to gain initial access, lateral movement, privilege escalation, and eventually full domain compromise. Once attackers breach the domain initially, they can harvest hashes and credentials of domain accounts and abuse their access rights to move laterally within the network. This process is repeated until the attacker leverages their access to compromise a domain admin or enterpise admin user. From there, attackers can dump the password hashes stored on the domain controller, or use any of the plethora of methods to establish persistent and privileged access on the domain.
