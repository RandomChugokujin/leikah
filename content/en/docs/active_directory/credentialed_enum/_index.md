---
title: Credentialed Enumeration
description: Get a full view of the domain after obtaining a set of credentials
categories: [Active Directory, Enumeration]
tags: [Active Directory, Enumeration]
weight: 2
---

After getting a first set of usable domain credentials, through methods such as password spraying or cracking a response captured through LLMNR poisoning, we can authenticate to permitted domain services, including LDAP, using supported methods such as Kerberos or NTLM. A captured NetNTLMv2 response alone is not a reusable password or NT hash. We can leverage this access to build a broader view of the domain, subject to our account's permissions and network reachability. We may be able to enumerate information such as:
- Users, computers, and groups
- Privileges and access rights
- Active Directory Certificate Services (AD CS) configuration
- Domain trust relationships
