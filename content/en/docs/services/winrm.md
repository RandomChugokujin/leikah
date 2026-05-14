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

The **Windows Remote Management (WinRM)** is a simple Windows integrated remote management protocol based on the command line. It uses **Simple Object Access Protocol (SOAP)** API over HTTP(S) to facilitate communication between the client and the server.

WinRM allows PowerShell commands to be executed on the server, which is why it is also referred to as PowerShell Remoting (PSRemote).

WinRM is a Windows feature that must be explicitly enabled.

## Service Enumeration
A Nmap Scan of TCP ports 5985 and 5986 will confirm whether WinRM is available from our attacker host.
```bash
sudo nmap -sVC <host> -p 5985,5986
```

{{% alert title="Note" %}}
If WinRM is available, Nmap will often report TCP port 5986 as closed even if it's open. If Nmap finds TCP port 5985 open and the service reported is `Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)`, you can be 99% sure WinRM is enabled on the target host.
{{% /alert %}}

We can also use NetExec to interact with WinRM.


### Remote Management Users
The **Remote Management Users** group in Windows have the privilege to use WinRM. In an Active Directory environment, local and domain users may be assigned to local Remote Management Users groups on individual machines, or the domain group with the same name, which have the ability to access WinRM on all machines on the domain.
- Administrator users are also allowed to use WinRM by default.

Local Remote Management users can only be enumerating using system commands such as `net localgroup "Remote Management Users"`. But they can be verified from a Linux attack machine using NetExec.

```shell-session
╭─brian@rx-93-nu ~
╰─$ nxc winrm 10.10.0.4 -u amuro.ray -p 'Password1'
WINRM       10.10.0.4       5985   RX-0-UNICORN     [*] Windows 11 / Server 2025 Build 26100 (name:RX-0-UNICORN) (domain:GUNDAM.local)
WINRM       10.10.0.4       5985   RX-0-UNICORN     [+] GUNDAM.local\amuro.ray:Password1 (Pwn3d!)
```

Members of the Domain **Remote Management Users** group can be queried against the domain controller:
```shell-session
╭─brian@rx-93-nu ~
╰─$ nxc ldap 10.10.0.3 -u Amuro.Ray -p Password1 --groups 'Remote Management Users'
LDAP        10.10.0.3       389    RA-CAILUM        [*] Windows 11 / Server 2025 Build 26100 (name:RA-CAILUM) (domain:GUNDAM.local) (signing:Enforced) (channel binding:When Supported)
LDAP        10.10.0.3       389    RA-CAILUM        [+] GUNDAM.local\Amuro.Ray:Password1
LDAP        10.10.0.3       389    RA-CAILUM        Char Aznable
```

Please check the sections on Windows Group enumeration or [Domain Group Enumeration]({{< relref "/docs/active_directory/credentialed_enum/user_enum/#domain-groups">}}) for more details.

## Service Interaction
Windows PowerShell cmdlet `Enter-PSSession` can be used to establish a PSRemote interactive session on a remote machine.
```shell-session
PS C:\> $password = ConvertTo-SecureString "Password1" -AsPlainText -Force
PS C:\> $cred = new-object System.Management.Automation.PSCredential ("GUNDAM\amuro.ray", $password)
PS C:\> Enter-PSSession -ComputerName RX-0-UNICORN -Credential $cred

[RX-0-UNICORN]: PS C:\Users\amuro.ray\Documents> hostname
RX-0-UNICORN
[RX-0-UNICORN]: PS C:\Users\amuro.ray\Documents> Exit-PSSession
```

From a Linux machine, `evil-winrm` can be used to establish PSRemote sessions.
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

NT hash can be used in lieu of a cleartext password with the `-H` option.
```bash
evil-winrm -i <host> -u <user> -H <NT_hash>
```

Pass-the-Ticket is also supported by `evil-winrm`, however, ensure that the target realm is configured correctly inside `/etc/krb5.conf`, and that the target's hostname can be resolved from your machine.
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
- `services`: list all services showing if there your account has permissions over each one, no admin privs required

For more advanced usage of `evil-winrm`, check out the Project's [GitHub Page](https://github.com/hackplayers/evil-winrm).
