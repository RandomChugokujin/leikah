---
title: Active Directory ACL Abuse
description: Abuse of ACL access rights to achieve lateral movement
categories: [Active Directory, Privilege Escalation]
tags: [Active Direcotory, Privilege Escalation, Lateral Movement, ACL Abuse]
weight: 2
---

Permissions in Active Directory are controlled through **Access Control Lists (ACL)**. Each security principal (user, group, process) has a corresponding ACL. ACLs define both who has access to which assets or resource, and what level of access they are granted. ACLs are made up of **Access Control Entries (ACE)** that explicity allow and/or deny users or groups from access.

If misconfigured, ACLs can be leveraged by attackers to achieve lateral movement or privilege escalation inside the domain. The abuse of ACL access rights are dependent on the specific access granted to the attacking user.
