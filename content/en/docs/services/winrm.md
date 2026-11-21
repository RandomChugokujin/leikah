---
title: WinRM
description: Windows Remote Management
categories: [Services]
tags: [Services, WinRM, Remote Management, Evil-WinRM]
weight: 2
---

## Service Info
- Name: Windows Remote Management
- Purpose: Remote management of Windows machines over the network
- Listening port: 5985/TCP, 5986/TCP (with TLS)
- OS: Windows

**Windows Remote Management (WinRM)** is Microsoft's implementation of WS-Management, a **SOAP**-based management protocol over HTTP(S). It provides a service and management APIs as well as command-line tools.

PowerShell remoting can use WinRM to execute commands remotely, but the terms are not interchangeable: WinRM also supports other management operations, and modern PowerShell also supports remoting over SSH.

Remote access requires a configured listener and firewall rules. Defaults depend on the Windows edition/version and policy; PowerShell remoting is enabled by default on Windows Server 2012 and later.

## Service Enumeration
An Nmap scan of TCP ports 5985 and 5986 identifies candidate WinRM listeners; an open port alone does not confirm WS-Management or authorization.
```bash
sudo nmap -sVC <host> -p 5985,5986
```

{{% alert title="Note" %}}
HTTP (5985) and HTTPS (5986) listeners are configured separately; HTTPS may simply be absent. A `Microsoft HTTPAPI httpd 2.0` banner is not unique to WinRM. Verify the WS-Management endpoint, normally `/wsman` (for example, `Test-WSMan <host>` from PowerShell); service availability and shell authorization are separate checks.
{{% /alert %}}

We can also use NetExec to interact with WinRM.


### Remote Management Users
Membership in **Administrators** or **Remote Management Users** may permit remote sessions, subject to the endpoint security descriptor and policy. Add local or domain accounts/groups to the appropriate group on each host. A same-named group in AD does **not** automatically grant WinRM access to every domain member.
- Administrator users are also allowed to use WinRM by default.

Enumerate local membership with `net localgroup "Remote Management Users"` or an authorized remote SAM/RPC query. A NetExec WinRM login checks usable access, not group membership.

```shell-session
╭─brian@rx-93-nu ~
╰─$ nxc winrm 10.10.0.4 -u amuro.ray -p 'Password1'
WINRM       10.10.0.4       5985   RX-0-UNICORN     [*] Windows 11 / Server 2025 Build 26100 (name:RX-0-UNICORN) (domain:GUNDAM.local)
WINRM       10.10.0.4       5985   RX-0-UNICORN     [+] GUNDAM.local\amuro.ray:Password1 (Pwn3d!)
```

An AD **Remote Management Users** group can be queried against a domain controller, if present; this does not enumerate the local group on every member host:
```shell-session
╭─brian@rx-93-nu ~
╰─$ nxc ldap 10.10.0.3 -u Amuro.Ray -p Password1 --groups 'Remote Management Users'
LDAP        10.10.0.3       389    RA-CAILUM        [*] Windows 11 / Server 2025 Build 26100 (name:RA-CAILUM) (domain:GUNDAM.local) (signing:Enforced) (channel binding:When Supported)
LDAP        10.10.0.3       389    RA-CAILUM        [+] GUNDAM.local\Amuro.Ray:Password1
LDAP        10.10.0.3       389    RA-CAILUM        Char Aznable
```

Please check the sections on Windows Group enumeration or [Domain Group Enumeration]({{< relref "/docs/active_directory/credentialed_enum/user_enum/#domain-groups">}}) for more details.

## Service Interaction
The Windows PowerShell cmdlet `Enter-PSSession` can be used to establish an interactive PowerShell remoting session on a remote machine.
```shell-session
PS C:\> $password = ConvertTo-SecureString "Password1" -AsPlainText -Force
PS C:\> $cred = new-object System.Management.Automation.PSCredential ("GUNDAM\amuro.ray", $password)
PS C:\> Enter-PSSession -ComputerName RX-0-UNICORN -Credential $cred

[RX-0-UNICORN]: PS C:\Users\amuro.ray\Documents> hostname
RX-0-UNICORN
[RX-0-UNICORN]: PS C:\Users\amuro.ray\Documents> Exit-PSSession
```

From a Linux machine, `evil-winrm` can be used to establish PowerShell remoting sessions.
```shell-session
╭─brian@rx-93-nu ~
╰─$ evil-winrm -i 10.10.0.4 -u amuro.ray -p 'Password1'

Evil-WinRM shell v3.9

Warning: Remote path completions is disabled due to ruby limitation: undefined method 'quoting_detection_proc' for module Reline

Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Amuro.Ray\Documents> hostname
RX-0-UNICORN
```

An NT hash can be used in lieu of a cleartext password with the `-H` option.
```bash
evil-winrm -i <host> -u <user> -H <NT_hash>
```

Pass-the-Ticket is also supported by `evil-winrm`. Ensure that a valid Kerberos ticket is available in the credential cache, the target realm is configured correctly in `/etc/krb5.conf`, and the target hostname resolves from your machine.
```shell-session
╭─brian@rx-93-nu ~
╰─$ evil-winrm -i rx-0-unicorn -r gundam.local

Evil-WinRM shell v3.9

Warning: Remote path completions is disabled due to ruby limitation: undefined method 'quoting_detection_proc' for module Reline

Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Amuro.Ray\Documents> hostname
RX-0-UNICORN
```

A lot of useful commands are provided by `evil-winrm`:
- `download <file>`: Download file from target machine
- `upload <file>`: Upload file from attacker machine
- `services`: list services and show whether your account has permissions over each one; no admin privileges required

For more advanced usage of `evil-winrm`, check out the project's [GitHub page](https://github.com/hackplayers/evil-winrm).
