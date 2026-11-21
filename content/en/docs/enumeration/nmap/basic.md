---
title: Nmap Basic Usage
description: Discover hosts and open ports with Nmap
categories: [Enumeration, Network Enumeration]
tags: [Enumeration, Nmap, Port Scanning, Host Discovery]
weight: 2
---

## Basic Scan
To begin a basic Nmap scan, simply provide it with the host(s) you wish to scan:
```bash
nmap <hosts>
```
The above command starts a port scan against the host(s) specified:
```shell-session
$ nmap 10.129.197.123
Starting Nmap 7.98 ( https://nmap.org ) at 2025-10-31 20:58 -0500
Nmap scan report for 10.129.197.123
Host is up (0.057s latency).
Not shown: 993 closed tcp ports (conn-refused)
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
110/tcp   open  pop3
139/tcp   open  netbios-ssn
143/tcp   open  imap
445/tcp   open  microsoft-ds
31337/tcp open  Elite

Nmap done: 1 IP address (1 host up) scanned in 1.88 seconds
```

The `<hosts>` argument can be:

- Individual IP addresses: `10.129.2.18 10.129.2.19 10.129.2.20`
- A range of IP addresses: `10.129.2.18-20`
- CIDR: `10.129.2.0/24`
- Hostnames: `example.com`

To have Nmap read the list of hosts from a file, use `-iL` to specify the filename. This example performs host discovery only (`-sn`):
```shell-session
$ cat hosts.txt
10.129.2.18
10.129.2.19
10.129.2.20
```
```bash
nmap -sn -iL hosts.txt
```

## Port Specification
To specify ports and ranges to scan, use the `-p` argument:
```bash
nmap -p <ports> <hosts>
```
The `-p` argument accepts:
- Individual port numbers: `80`, `22,80`
- Ranges of ports: `1-1000`
- A combination of both: `22,80,100-500`

For a complete scan of ports 1-65535 in the selected protocol, use the `-p-` shorthand (TCP by default).
```bash
nmap -p- <hosts>
```

Alternatively, use `--top-ports` to specify the number of most common ports to scan. By default, Nmap scans the 1000 most common ports per selected protocol.
```bash
nmap --top-ports <number> <hosts>
```

With the standard `nmap-services` frequency data, `-F` scans the 100 most common ports per selected protocol, like `--top-ports 100`.
```bash
nmap -F <hosts>
```
### Port Scanning without Ping Probes

Nmap normally performs host discovery before beginning a port scan. These probes may use ICMP, TCP, or local-network ARP/IPv6 Neighbor Discovery, depending on privileges and the network. Firewalls may block the probes even when services are reachable, causing Nmap to conclude that the host is not up.
```shell-session
$ nmap 10.10.65.55
Starting Nmap 7.98 ( https://nmap.org ) at 2025-10-27 20:58 -0500
Note: Host seems down. If it is really up, but blocking our ping probes, try -Pn
Nmap done: 1 IP address (0 hosts up) scanned in 3.02 seconds
```
As its output suggests, we can re-scan the host with `-Pn`, which skips normal host discovery and attempts to scan every specified host. Local Ethernet targets still normally require ARP/Neighbor Discovery.
```shell-session
$ nmap 10.10.65.55 -Pn
Starting Nmap 7.98 ( https://nmap.org ) at 2025-10-27 21:23 -0500
Nmap scan report for 10.10.65.55
Host is up (0.15s latency).
Not shown: 986 filtered tcp ports (no-response)
PORT     STATE SERVICE
53/tcp   open  domain
88/tcp   open  kerberos-sec
135/tcp  open  msrpc
139/tcp  open  netbios-ssn
389/tcp  open  ldap
445/tcp  open  microsoft-ds
464/tcp  open  kpasswd5
593/tcp  open  http-rpc-epmap
636/tcp  open  ldapssl
3268/tcp open  globalcatLDAP
3269/tcp open  globalcatLDAPssl
3389/tcp open  ms-wbt-server
5357/tcp open  wsdapi
5985/tcp open  wsman

Nmap done: 1 IP address (1 host up) scanned in 12.92 seconds
```

### Verbose Output
Use `-v`/`-vv` flags to increase the verbosity of Nmap's output, which shows us open ports directly when Nmap detects them.
```shell-session
$ sudo nmap 10.129.2.28 -p- -sV -v

Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-15 20:03 CEST
NSE: Loaded 45 scripts for scanning.
Initiating ARP Ping Scan at 20:03
Scanning 10.129.2.28 [1 port]
Completed ARP Ping Scan at 20:03, 0.03s elapsed (1 total hosts)
Initiating Parallel DNS resolution of 1 host. at 20:03
Completed Parallel DNS resolution of 1 host. at 20:03, 0.02s elapsed
Initiating SYN Stealth Scan at 20:03
Scanning 10.129.2.28 [65535 ports]
Discovered open port 995/tcp on 10.129.2.28
Discovered open port 80/tcp on 10.129.2.28
Discovered open port 993/tcp on 10.129.2.28
Discovered open port 143/tcp on 10.129.2.28
Discovered open port 25/tcp on 10.129.2.28
Discovered open port 110/tcp on 10.129.2.28
Discovered open port 22/tcp on 10.129.2.28
<SNIP>
```

## Host Discovery
Use `-sn` to disable port scanning and perform host discovery against the specified hosts. This is not limited to ICMP ping.
```bash
nmap -sn <hosts>
```

## Performance Tuning
Nmap gives 6 templates to tune the aggressiveness of our scans, from 0 being the slowest to 5 being the fastest. However, a more aggressive profile could cause more false negatives as it sets shorter timeouts for the host to respond.

Choose a timing template with `-T`:
- `-T0` / `-T paranoid`
- `-T1` / `-T sneaky`
- `-T2` / `-T polite`
- `-T3` / `-T normal`
- `-T4` / `-T aggressive`
- `-T5` / `-T insane`

By default, Nmap uses `-T3`. But for certification exams and CTFs, `-T4` is a good balance between speed and consistency.

