---
title: Active Directory Credentialed Enumeration
description: Get a full view of the domain after obtaining a set a credentials
categories: [Active Directory, Enumeration]
tags: [Active Direcotory, Enumeration]
weight: 2
---

After getting a first set of credentials, through methods such as password spraying, LLMNR Poisoning and etc., we now have access to the core services on the domain, including Kerberos and NTLM authentication, as well as LDAP. We can now leverage this access to get a full view of the domain. We will be able to enumerate information such as:
- Users, computers, and groups
- Privileges and access rights
- Active Directory Certificate Service (ADCS) configuration
- Domain trust relationships
