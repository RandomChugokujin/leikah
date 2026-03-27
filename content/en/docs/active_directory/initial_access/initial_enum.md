---
title: Initial Enumeration
description: Enumeration of AD domain without credentials
categories: [Active Directory]
tags: [Active Direcotory, Initial Access, Enumeration]
weight: 2
---

The information we can glean without a set of domain credentials are limited. We can use network enumeration techniques to identify active hosts on the network, enumerate the services running via port scanning, and get a partial list of domain users. Keep in mind that some methods below, epecially those that interact with the target hosts directly, can create noise in the target network. They should be avoided if stealth is a concern for the engagement.

We assume we are positioned on a machine directly connected to the target network running Active Directory.

## Passive Host Identification
First, we may find some hosts on the network by listening on the network. We may use **Wireshark** to capture and inspect packets, or if GUI is not available, we can use command-line utilities such as `tcpdump` to save output to a pcap file, transfer the pacp file to another machine, and analyze it offline.
```bash
sudo tcpdump -i <iface>
```
Particularly, we want to pay attention to **ARP** and **LLMNR/NBNS/MDNS** packets, as the former reveals IP address, and the latter reveals IP address associations with hostnames.

Alternatively, **Responder**'s analysis mode can be used to lisen for **LLMNR/NBNS/MDNS** requests and responses without poisoning them.
```bash
sudo responder -I ens224 -A
```

## Active Host Identification
We can do a quick ICMP sweep of the subnet using `fping`, which can issue ICMP ping requests to a list of multiple hosts at once.

> Note that many Windows hosts, especially workstation editiions (Windows 11, Windows 10, etc.) may be configured to not to respond to ping requests by default.

```bash
fping -asgq 10.10.0.0/24
```
- `-a` for showing alive hosts.
- `-s` for printing cumulative stats upon exit.
- `-g` for generating a list of host from the CIDR network notation specified
- `-q` for quiet output, hiding per-probe results

## Port Scanning and Service Enumeration
Now, we may use Nmap to scan the ports of the active hosts to find the services available on them.
```bash
sudo nmap -v -A -iL <host_list> -oN <output_filename>
```

Besides just identifying the services running, Nmap's default scripts will also enumerate the hosts' hostnames, the name of the domain it belongs to, and much more.
```shell-session
╭─brian@rx-93-nu ~
╰─$ sudo nmap -A 10.10.0.3 -T4
[sudo] password for brian:
Starting Nmap 7.98 ( https://nmap.org ) at 2026-03-27 16:48 -0500
Nmap scan report for gundam.local (10.10.0.3)
Host is up (0.0018s latency).
Not shown: 985 filtered tcp ports (no-response)
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-03-27 21:48:21Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: GUNDAM.local, Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=RA-CAILUM.GUNDAM.local
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:RA-CAILUM.GUNDAM.local
| Not valid before: 2025-06-06T04:55:25
|_Not valid after:  2026-06-06T04:55:25
|_ssl-date: TLS randomness does not represent time
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=RA-CAILUM.GUNDAM.local
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:RA-CAILUM.GUNDAM.local
| Not valid before: 2025-06-06T04:55:25
|_Not valid after:  2026-06-06T04:55:25
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: GUNDAM.local, Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=RA-CAILUM.GUNDAM.local
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:RA-CAILUM.GUNDAM.local
| Not valid before: 2025-06-06T04:55:25
|_Not valid after:  2026-06-06T04:55:25
|_ssl-date: TLS randomness does not represent time
3269/tcp open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: GUNDAM.local, Site: Default-First-Site-Name)
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=RA-CAILUM.GUNDAM.local
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:RA-CAILUM.GUNDAM.local
| Not valid before: 2025-06-06T04:55:25
|_Not valid after:  2026-06-06T04:55:25
3389/tcp open  ms-wbt-server
| rdp-ntlm-info:
|   Target_Name: GUNDAM
|   NetBIOS_Domain_Name: GUNDAM
|   NetBIOS_Computer_Name: RA-CAILUM
|   DNS_Domain_Name: GUNDAM.local
|   DNS_Computer_Name: RA-CAILUM.GUNDAM.local
|   DNS_Tree_Name: GUNDAM.local
|   Product_Version: 10.0.26100
|_  System_Time: 2026-03-27T21:49:04+00:00
| ssl-cert: Subject: commonName=RA-CAILUM.GUNDAM.local
| Not valid before: 2025-12-09T17:46:23
|_Not valid after:  2026-06-10T17:46:23
|_ssl-date: TLS randomness does not represent time
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port3389-TCP:V=7.98%I=7%D=3/27%Time=69C6FB2A%P=x86_64-pc-linux-gnu%r(Te
SF:rminalServerCookie,13,"\x03\0\0\x13\x0e\xd0\0\0\x124\0\x02\?\x08\0\x02\
SF:0\0\0");
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Microsoft Windows 2022|2016|11 (96%)
OS CPE: cpe:/o:microsoft:windows_server_2022 cpe:/o:microsoft:windows_server_2016 cpe:/o:microsoft:windows_11
Aggressive OS guesses: Microsoft Windows Server 2022 (96%), Microsoft Windows Server 2016 (91%), Microsoft Windows 11 21H2 (90%)
No exact OS matches for host (test conditions non-ideal).
Service Info: Host: RA-CAILUM; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time:
|   date: 2026-03-27T21:49:06
|_  start_date: N/A
| smb2-security-mode:
|   3.1.1:
|_    Message signing enabled but not required

TRACEROUTE
HOP RTT     ADDRESS
1   1.79 ms gundam.local (10.10.0.3)

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 95.06 seconds
```
Domain controllers usually have several services that are crucial for maintaining the Active Directory network:
- DNS (53/TCP)
- Kerberos (88/TCP, 464/TCP)
- LDAP (389/TCP, 636/TCP, 3268/TCP, 3269/TCP)
- MSRPC (135/TCP), NetBIOS (139/TCP), SMB (445/TCP)

There are also common remote management services that may be exposed on domain controllers or any other hosts on the network:
- RDP (3389/TCP)
- WinRM (5985/TCP)

Other services may include MSSQL (1433/TCP) or web servers (80/TCP, 443/TCP).

For more details on Nmap usage, please see the articles in the [Nmap section]({{< relref "/docs/enumeration/nmap/">}}).

## User Enumeration
We can passively enumerate users via OSINT. We can browse the target organization's website and social media for employee names and emails. We should pay attention to the user naming convention that the organization employs. Below are a few common ones:
- FirstInitialLastname (`John Smith` -> `jsmith`)
- Firstname.LastName (`John Smith` -> `john.smith`)

We can actively enumerate users on the domain, even if we don't have any credentials on the domain, using `Kerbrute`, which enumerates users through Kerberos pre-authentication. This is considered a stealthier approach since Kerberos pre-auth failures doesn't generate logs by default.
```bash
kerbrute userenum -d <domain_name> --dc <DC_IP> <username_wordlist> -o <output_file>
```

For the potential username wordlist we provide, we can create our own wordlist with the results of our OSINT, or use this [statistically likely list of usernames](https://github.com/insidetrust/statistically-likely-usernames).
