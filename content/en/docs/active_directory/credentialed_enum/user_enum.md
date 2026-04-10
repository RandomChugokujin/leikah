---
title: Domain User and Group Enumeration
description: Enumerate users and groups within an Active Directory domain
categories: [Active Directory, Enumeration]
tags: [Active Direcotory, Enumeration]
weight: 2
---

With the ability to authenticate to an Active Directory domain, we can now get a full list of users and groups on the domain. This can be helpful for us to plan further attacks and expand our access within the domain.

## Domain Users

### Linux Perspective
The `--users` option may be used with **NetExec** to enumerate domain users. Note we have to use protocol `ldap` and our target must be a domain controller.
```bash
nxc ldap <dc_host> -u <username> -p <password> --users
```
NetExec presents us with a list of users, when their password is last set, number of failed login attempts (for password spraying), as well as the description field.
```shell-session
╭─brian@rx-93-nu ~
╰─$ nxc ldap 10.10.0.3 -u amuro.ray -p 'Password1' --users
LDAP        10.10.0.3       389    RA-CAILUM        [*] Windows 11 / Server 2025 Build 26100 (name:RA-CAILUM) (domain:GUNDAM.local) (signing:Enforced) (channel binding:When Supported)
LDAP        10.10.0.3       389    RA-CAILUM        [+] GUNDAM.local\amuro.ray:Password1
LDAP        10.10.0.3       389    RA-CAILUM        [*] Enumerated 7 domain users: GUNDAM.local
LDAP        10.10.0.3       389    RA-CAILUM        -Username-                    -Last PW Set-       -BadPW-  -Description-
LDAP        10.10.0.3       389    RA-CAILUM        Administrator                 2025-06-21 11:53:50 0        Built-in account for administering the computer/domain
LDAP        10.10.0.3       389    RA-CAILUM        Guest                         <never>             0        Built-in account for guest access to the computer/domain
LDAP        10.10.0.3       389    RA-CAILUM        krbtgt                        2025-06-05 19:31:16 0        Key Distribution Center Service Account
LDAP        10.10.0.3       389    RA-CAILUM        amuro.ray                     2025-06-06 00:10:46 0
LDAP        10.10.0.3       389    RA-CAILUM        Char.Aznable                  2025-06-06 00:11:44 0
LDAP        10.10.0.3       389    RA-CAILUM        svc_sql                       2025-06-06 00:13:47 0        Password: Qwerty123
LDAP        10.10.0.3       389    RA-CAILUM        Bright.Noa                    2025-06-21 11:58:21 0
```
[Windapsearch](https://github.com/ropnop/windapsearch/blob/master/windapsearch.py) can query the domain controller for all domain users via LDAP. The `-U` option queries for all objects where `objectCategory=user`.
```bash
windapsearch.py -d <domain_fqdn> --dc-ip <dc_ip> -u <DOMAIN>\\<username> -p <password> -U
```
- `<DOMAIN>` specified within `-u` should not include TLD (`GUNDAM.LOCAL` -> `GUNDAM\\amuro.ray`)

### Windows Perspective
The `Get-ADUser` cmdlet from the built-in Active Directory PowerShell module can be used to enumerate domain users from a Windows machine.
```shell-session
PS C:\Users\Administrator> Get-ADUser -Filter * | select SamAccountName

SamAccountName
--------------
Administrator
Guest
krbtgt
amuro.ray
Char.Aznable
svc_sql
Bright.Noa
```
The `net user` command may be used as well.
```shell-session
PS C:\Users\Administrator> net user /domain

User accounts for \\RA-CAILUM

-------------------------------------------------------------------------------
Administrator            amuro.ray                Bright.Noa
Char.Aznable             Guest                    krbtgt
svc_sql
The command completed successfully.
```

More detailed information may be obtained using the `Get-DomainUser` function from PowerView.
```shell-session
PS C:\research> Import-Module .\PowerView.ps1
PS C:\research> Get-DomainUser -Identity amuro.ray -Domain gundam.local | Select-Object -Property name,samaccountname,description,memberof,whencreated,pwdlastset,lastlogontimestamp,accountexpires,admincount,userprincipalname,serviceprincipalname,useraccountcontrol


name                 : Amuro Ray
samaccountname       : amuro.ray
description          :
memberof             :
whencreated          : 6/6/2025 5:10:46 AM
pwdlastset           : 6/6/2025 12:10:46 AM
lastlogontimestamp   : 4/7/2026 12:59:11 PM
accountexpires       : NEVER
admincount           :
userprincipalname    : amuro.ray@GUNDAM.local
serviceprincipalname :
useraccountcontrol   : NORMAL_ACCOUNT, DONT_EXPIRE_PASSWORD
```

#### Testing Local Admin Access
Using PowerView function `Test-AdminAccess`, we may test if our current user is a local administrator on the local machine or a remote one.
```shell-session
PS C:\research> Test-AdminAccess -ComputerName RX-0-UNICORN

ComputerName IsAdmin
------------ -------
RX-0-UNICORN    True

```

## Domain Groups

### Linux Perspective
NetExec can be used with option `--groups` via LDAP protocol to enumerate domain groups.
```bash
nxc ldap <dc_host> -u <username> -p <password> --groups
```
```shell-session
╭─brian@rx-93-nu ~
╰─$ nxc ldap 10.10.0.3 -u amuro.ray -p 'Password1' --groups
LDAP        10.10.0.3       389    RA-CAILUM        [*] Windows 11 / Server 2025 Build 26100 (name:RA-CAILUM) (domain:GUNDAM.local) (signing:Enforced) (channel binding:When Supported)
LDAP        10.10.0.3       389    RA-CAILUM        [+] GUNDAM.local\amuro.ray:Password1
LDAP        10.10.0.3       389    RA-CAILUM        -Group-                                  -Members- -Description-
LDAP        10.10.0.3       389    RA-CAILUM        Administrators                           4         Administrators have complete and unrestricted access to the computer/domain
LDAP        10.10.0.3       389    RA-CAILUM        Users                                    3         Users are prevented from making accidental or intentional system-wide changes and can run most applications
LDAP        10.10.0.3       389    RA-CAILUM        Guests                                   2         Guests have the same access as members of the Users group by default, except for the
Guest account which is further restricted
LDAP        10.10.0.3       389    RA-CAILUM        Print Operators                          0         Members can administer printers installed on domain controllers
LDAP        10.10.0.3       389    RA-CAILUM        Backup Operators                         0         Backup Operators can override security restrictions for the sole purpose of backing up or restoring files
[...]
LDAP        10.10.0.3       389    RA-CAILUM        Cert Admins                              1
LDAP        10.10.0.3       389    RA-CAILUM        SQLServer2005SQLBrowserUser$RA-CAILUM    0         Members in the group have the required access and privileges to be assigned as the log on account for the associated instance of SQL Server Browser.
```
If we specify the name of a group after the option, we can get a list of members within that group.
```bash
nxc ldap <dc_host> -u <username> -p <password> --groups <group>
```
```shell-session
╭─brian@rx-93-nu ~
╰─$ nxc ldap 10.10.0.3 -u amuro.ray -p 'Password1' --groups 'Cert Admins'
LDAP        10.10.0.3       389    RA-CAILUM        [*] Windows 11 / Server 2025 Build 26100 (name:RA-CAILUM) (domain:GUNDAM.local) (signing:Enforced) (channel binding:When Supported)
LDAP        10.10.0.3       389    RA-CAILUM        [+] GUNDAM.local\amuro.ray:Password1
LDAP        10.10.0.3       389    RA-CAILUM        Bright Noa
```

Using the `-G` option with [Windapsearch](https://github.com/ropnop/windapsearch) enumerates domain groups, while `-m` option enumerates members of a specific group.
```bash
windapsearch.py -d <domain_fqdn> --dc-ip <dc_ip> -u <DOMAIN>\\<username> -p <password> -G
windapsearch.py -d <domain_fqdn> --dc-ip <dc_ip> -u <DOMAIN>\\<username> -p <password> -m <group>
```

### Windows Perspective
The native `Get-ADGroup` cmdlet may be used to get a list of domain groups.
```shell-session
PS C:\research> Get-ADGroup -Filter * | select name

name
----
Administrators
Users
Guests
Print Operators
Backup Operators
Replicator
Remote Desktop Users
Network Configuration Operators
Performance Monitor Users
Performance Log Users
[...]
```
Individual groups may be enumerated with `-Identity` option.
```shell-session
PS C:\research> Get-ADGroup -Identity Administrators


DistinguishedName : CN=Administrators,CN=Builtin,DC=GUNDAM,DC=local
GroupCategory     : Security
GroupScope        : DomainLocal
Name              : Administrators
ObjectClass       : group
ObjectGUID        : 81039797-5691-454c-be37-268a5b3e7cbd
SamAccountName    : Administrators
SID               : S-1-5-32-544
```

A list of domain groups can also be obtained using `Get-DomainGroups` function from PowerView.
```shell-session
PS C:\research> Get-DomainGroup | select name

name
----
Administrators
Users
Guests
Print Operators
Backup Operators
Replicator
Remote Desktop Users
Network Configuration Operators
Performance Monitor Users
Performance Log Users
[...]
```

To list out members to a group, we use `Get-ADGroupMember` cmdlet:
```shell-session
PS C:\research> Get-ADGroupMember -Identity Administrators


distinguishedName : CN=SQL Service,CN=Users,DC=GUNDAM,DC=local
name              : SQL Service
objectClass       : user
objectGUID        : 00a6b75d-cae4-4993-960d-f74b18b0b603
SamAccountName    : svc_sql
SID               : S-1-5-21-790304770-1385196242-1780550448-1105

distinguishedName : CN=Domain Admins,OU=Security Groups,DC=GUNDAM,DC=local
name              : Domain Admins
objectClass       : group
objectGUID        : 31b2e9ce-edfd-4de0-9123-c90f0dbfdcfd
SamAccountName    : Domain Admins
SID               : S-1-5-21-790304770-1385196242-1780550448-512

distinguishedName : CN=Enterprise Admins,OU=Security Groups,DC=GUNDAM,DC=local
name              : Enterprise Admins
objectClass       : group
objectGUID        : e7e43411-7fa8-4eab-8755-eae42aca3b61
SamAccountName    : Enterprise Admins
SID               : S-1-5-21-790304770-1385196242-1780550448-519

distinguishedName : CN=Administrator,CN=Users,DC=GUNDAM,DC=local
name              : Administrator
objectClass       : user
objectGUID        : 0646e2a6-ed78-46df-b79e-cd93409f29b3
SamAccountName    : Administrator
SID               : S-1-5-21-790304770-1385196242-1780550448-500
```

The PowerView function `Get-DomainGroupMember` achives the above, with the added ability to unroll nested group memberships when used with `-Recurse` option.
```shell-session
PS C:\research> Get-DomainGroupMember -Identity "Domain Admins" -Recurse


GroupDomain             : GUNDAM.local
GroupName               : Domain Admins
GroupDistinguishedName  : CN=Domain Admins,OU=Security Groups,DC=GUNDAM,DC=local
MemberDomain            : GUNDAM.local
MemberName              : svc_sql
MemberDistinguishedName : CN=SQL Service,CN=Users,DC=GUNDAM,DC=local
MemberObjectClass       : user
MemberSID               : S-1-5-21-790304770-1385196242-1780550448-1105

GroupDomain             : GUNDAM.local
GroupName               : Domain Admins
GroupDistinguishedName  : CN=Domain Admins,OU=Security Groups,DC=GUNDAM,DC=local
MemberDomain            : GUNDAM.local
MemberName              : Administrator
MemberDistinguishedName : CN=Administrator,CN=Users,DC=GUNDAM,DC=local
MemberObjectClass       : user
MemberSID               : S-1-5-21-790304770-1385196242-1780550448-500
```


## Domain Computers
Computer accounts are special user accounts that is assigned to each domain-joined computer for it to participate in the domain. They can be enumerated using the `--computers` option with NetExec.
```bash
nxc ldap <dc_host> -u <username> -p <password> --users
```
```shell-session
╭─brian@rx-93-nu ~
╰─$ nxc ldap 10.10.0.3 -u amuro.ray -p 'Password1' --computers
LDAP        10.10.0.3       389    RA-CAILUM        [*] Windows 11 / Server 2025 Build 26100 (name:RA-CAILUM) (domain:GUNDAM.local) (signing:Enforced) (channel binding:When Supported)
LDAP        10.10.0.3       389    RA-CAILUM        [+] GUNDAM.local\amuro.ray:Password1
LDAP        10.10.0.3       389    RA-CAILUM        [*] Total records returned: 5
LDAP        10.10.0.3       389    RA-CAILUM        RA-CAILUM$
LDAP        10.10.0.3       389    RA-CAILUM        RX-0-UNICORN$
LDAP        10.10.0.3       389    RA-CAILUM        MSN-04-SAZABI$
LDAP        10.10.0.3       389    RA-CAILUM        SINANJU$
LDAP        10.10.0.3       389    RA-CAILUM        MSZ-006-ZETA$
```

From the Windows Perspective, `Get-DomainComputer` may be used to find comptuer accounts.
```shell-session
PS C:\research> Get-DomainComputer | select samaccountname

samaccountname
--------------
RA-CAILUM$
RX-0-UNICORN$
MSN-04-SAZABI$
SINANJU$
MSZ-006-ZETA$
```

## RID Cycling
RID Cycling is a technique used to enumerate users and groups on Windows systems. Every account (users and groups) have a **Security Identifier (SID)** that looks like the following:
```txt
S-1-5-21-<domain identifier>-RID
```
The `RID` portion (Relative Identifier) uniquely identifies a objects within a domain.

RID Cycling cycles through ranges of valid RIDs to enumerate valid users and accounts.

> The number of users and groups discovered depend on the maximum and minimum RID values being cycled. This is especially true within very large organizations.

We can use `lookupsid.py` from Impacket for this purpose.
```bash
lookupsid.py <user>:<password>@<host> [<max_sid>]
```
```shell-session
╭─brian@rx-93-nu ~
╰─$ lookupsid.py amuro.ray:Password1@10.10.0.3
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies

[*] Brute forcing SIDs at 10.10.0.3
[*] StringBinding ncacn_np:10.10.0.3[\pipe\lsarpc]
[*] Domain SID is: S-1-5-21-790304770-1385196242-1780550448
498: GUNDAM\Enterprise Read-only Domain Controllers (SidTypeGroup)
500: GUNDAM\Administrator (SidTypeUser)
501: GUNDAM\Guest (SidTypeUser)
502: GUNDAM\krbtgt (SidTypeUser)
512: GUNDAM\Domain Admins (SidTypeGroup)
513: GUNDAM\Domain Users (SidTypeGroup)
514: GUNDAM\Domain Guests (SidTypeGroup)
515: GUNDAM\Domain Computers (SidTypeGroup)
516: GUNDAM\Domain Controllers (SidTypeGroup)
517: GUNDAM\Cert Publishers (SidTypeAlias)
518: GUNDAM\Schema Admins (SidTypeGroup)
519: GUNDAM\Enterprise Admins (SidTypeGroup)
520: GUNDAM\Group Policy Creator Owners (SidTypeGroup)
521: GUNDAM\Read-only Domain Controllers (SidTypeGroup)
522: GUNDAM\Cloneable Domain Controllers (SidTypeGroup)
525: GUNDAM\Protected Users (SidTypeGroup)
526: GUNDAM\Key Admins (SidTypeGroup)
527: GUNDAM\Enterprise Key Admins (SidTypeGroup)
528: GUNDAM\Forest Trust Accounts (SidTypeGroup)
529: GUNDAM\External Trust Accounts (SidTypeGroup)
553: GUNDAM\RAS and IAS Servers (SidTypeAlias)
571: GUNDAM\Allowed RODC Password Replication Group (SidTypeAlias)
572: GUNDAM\Denied RODC Password Replication Group (SidTypeAlias)
1000: GUNDAM\RA-CAILUM$ (SidTypeUser)
1101: GUNDAM\DnsAdmins (SidTypeAlias)
1102: GUNDAM\DnsUpdateProxy (SidTypeGroup)
1103: GUNDAM\amuro.ray (SidTypeUser)
1104: GUNDAM\Char.Aznable (SidTypeUser)
1105: GUNDAM\svc_sql (SidTypeUser)
1106: GUNDAM\RX-0-UNICORN$ (SidTypeUser)
1107: GUNDAM\MSN-04-SAZABI$ (SidTypeUser)
1109: GUNDAM\Bright.Noa (SidTypeUser)
1110: GUNDAM\Cert Admins (SidTypeGroup)
1115: GUNDAM\SINANJU$ (SidTypeUser)
1124: GUNDAM\MSZ-006-ZETA$ (SidTypeUser)
1125: GUNDAM\SQLServer2005SQLBrowserUser$RA-CAILUM (SidTypeAlias)
```

NetExec also have option `--rid-brute` for us to perform the same enumeration technique.
```bash
nxc smb <host> -u <username> -p <password> --rid-brute
```
```shell-session
╭─brian@rx-93-nu ~
╰─$ nxc smb 10.10.0.3 -u amuro.ray -p 'Password1' --rid-brute
SMB         10.10.0.3       445    RA-CAILUM        [*] Windows 11 / Server 2025 Build 26100 x64 (name:RA-CAILUM) (domain:GUNDAM.local) (signing:False) (SMBv1:None)
SMB         10.10.0.3       445    RA-CAILUM        [+] GUNDAM.local\amuro.ray:Password1
SMB         10.10.0.3       445    RA-CAILUM        498: GUNDAM\Enterprise Read-only Domain Controllers (SidTypeGroup)
SMB         10.10.0.3       445    RA-CAILUM        500: GUNDAM\Administrator (SidTypeUser)
SMB         10.10.0.3       445    RA-CAILUM        501: GUNDAM\Guest (SidTypeUser)
SMB         10.10.0.3       445    RA-CAILUM        502: GUNDAM\krbtgt (SidTypeUser)
SMB         10.10.0.3       445    RA-CAILUM        512: GUNDAM\Domain Admins (SidTypeGroup)
[...]
```

> Tip: we can grep for `SidTypeUser` if we only want a list of users or `SidTypeGroup` for a list of groups.

## SPN Users
Identifying service accounts, i.e. accounts with one or more **Service Principal Names (SPNs**) help us find opportunities for attack such as Kerberoasting or Silver Ticket.

We may use `GetUserSPNs.py` without the `-request` option to get a list of service accounts and their SPNs.
```bash
GetUserSPNs.py -dc-ip <dc_ip> <domain_fqdn>/<username>:<password>
```
```shell-session
╭─brian@rx-93-nu ~
╰─$ GetUserSPNs.py -dc-ip 10.10.0.3 gundam.local/amuro.ray:Password1
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies

ServicePrincipalName                        Name     MemberOf                                                              PasswordLastSet             LastLogon                   Delegation
------------------------------------------  -------  --------------------------------------------------------------------  --------------------------  --------------------------  ----------
MSSQLSvc/RA-CAILUM.GUNDAM.local:1433        svc_sql  CN=Group Policy Creator Owners,OU=Security Groups,DC=GUNDAM,DC=local  2025-06-06 00:13:47.605590  2026-04-07 16:16:56.632421

MSSQLSvc/RA-CAILUM.GUNDAM.local:SQLEXPRESS  svc_sql  CN=Group Policy Creator Owners,OU=Security Groups,DC=GUNDAM,DC=local  2025-06-06 00:13:47.605590  2026-04-07 16:16:56.632421
```
Alternatively, windapsearch with option `--user-spns` may also be used to retrieve a list of accounts with SPNs via LDAP.
```bash
windapsearch.py -d <domain_fqdn> --dc-ip <dc_ip> -u <DOMAIN>\\<username> -p <password> --user-spns
```

From the Windows Perspective, `Get-ADUser` may be used with filter `ServicePrincipalName -ne "$null"` to obtain SPN accounts.
```powershell
Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName
```

Alternatively, the `Get-DomainUser` function from PowerView may be used with `-SPN` option.
```powershell
Get-DomainUser -SPN | select samaccountname,serviceprincipalname
```

## Users with Kerberos Pre-Authentication Disabled
There is an option to disable requirement for Kerberos Pre-Authentication inside the UAC options for users within Active Directory. The user is thus vulnerable to AS-REProasting. We can identify AS-REProastable users with `GetNPUsers.py` without the `-request` option from Impacket on a Linux Machine.
```shell-session
╭─brian@rx-93-nu ~
╰─$ GetNPUsers.py -dc-ip 10.10.0.3 GUNDAM.LOCAL/amuro.ray:Password1
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies

Name          MemberOf  PasswordLastSet             LastLogon  UAC
------------  --------  --------------------------  ---------  --------
hathaway.noa            2026-04-09 20:36:53.519673  <never>    0x410200
```

From a Windows Machine, this can be identified using the `-PreauthNotRequired` option for `Get-DomainUser` function from PowerView.
```shell-session
PS C:\research> Get-DomainUser -PreauthNotRequired | select samaccountname,userprincipalname,useraccountcontrol | fl


samaccountname     : hathaway.noa
userprincipalname  : hathaway.noa@GUNDAM.local
useraccountcontrol : NORMAL_ACCOUNT, DONT_EXPIRE_PASSWORD, DONT_REQ_PREAUTH
```

## Logged on Users
If our user have admin privileges on the target (indicated by NetExec with the yellow `Pwn3d!` marker), we can enumerate logged on users using option `--loggedon-users`
```bash
nxc smb <host> -u <username> -p <password> --loggedon-users
```
