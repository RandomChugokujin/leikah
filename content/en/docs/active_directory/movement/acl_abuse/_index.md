---
title: ACL Abuse
description: Abuse of ACL access rights to achieve lateral movement
categories: [Active Directory, Privilege Escalation]
tags: [Active Directory, Privilege Escalation, Lateral Movement, ACL Abuse]
weight: 2
---

Permissions in Active Directory are controlled through **Access Control Lists (ACLs)** in object security descriptors. A discretionary ACL (DACL) defines which security principals can access an object and what level of access they are granted. DACLs are made up of **Access Control Entries (ACEs)** that explicitly allow or deny access rights.

If misconfigured, ACLs can be leveraged by attackers to achieve lateral movement or privilege escalation inside the domain. The abuse of ACL access rights is dependent on the specific access granted to the attacking user.
