---
title: Nmap Service and Host Enumeration
description: Footprint network services and the hosts running them
categories: [Enumeration, Network Enumeration]
tags: [Enumeration, Nmap, Port Scanning, Host Discovery, Serice Enumeration, NSE]
weight: 2
---

Although there is a convention for the port number of common services, we should strive to more accurately identify the services running instead of just taking guesses. Nmap can helps us by performing service numeration on open ports.

## Nmap Service Enumeration
Use the `-sV` flag to tell Nmap to perform Service enumeration on each of the ports it detects to be open:
```bash
nmap -sV <hosts>
```

Nmap's service enumeration attempts to give us with the type and version of service running.
```shell-session
$ sudo nmap 10.129.2.28 -p- -sV

Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-15 20:00 CEST
Nmap scan report for 10.129.2.28
Host is up (0.013s latency).
Not shown: 65525 closed ports
PORT      STATE    SERVICE      VERSION
22/tcp    open     ssh          OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
25/tcp    open     smtp         Postfix smtpd
80/tcp    open     http         Apache httpd 2.4.29 ((Ubuntu))
110/tcp   open     pop3         Dovecot pop3d
139/tcp   filtered netbios-ssn
143/tcp   open     imap         Dovecot imapd (Ubuntu)
445/tcp   filtered microsoft-ds
993/tcp   open     ssl/imap     Dovecot imapd (Ubuntu)
995/tcp   open     ssl/pop3     Dovecot pop3d
MAC Address: DE:AD:00:00:BE:EF (Intel Corporate)
Service Info: Host:  inlane; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 91.73 seconds
```
Nmap Service Enumeration relies on two mechanisms:
- **Banner Grabbing**: Nmap establishes a connection to the service and wait for it to present it with its banner, which often contains service information like type and version.
- **Service Signature Footprinting**: In the case that Nmap doesn't receive a banner within the timeout limit, it conducts footprinting against the service and analyzes the signature of its response. This makes the service enumeration process much longer.

## Manual Banner Grabbing
There are times where Nmap may be unable to enumerate the service type and version. We can manually grab the banner by connecting to the service with Netcat:
```shell-session
$ nc -nv 10.129.2.28 25

Connection to 10.129.2.28 port 25 [tcp/*] succeeded!
220 inlane ESMTP Postfix (Ubuntu)
```

## Nmap Script Scanning
Nmap also provides scripting capabilities with its **Nmap Scripting Engine (NSE)**. Nmap includes a series of scripts when you install it. They are stored under `/usr/share/nmap/scripts/`

```shell-session
$ ls -l /usr/share/nmap/scripts
total 5024
-rw-r--r-- 1 root root  3901 Sep 29 02:24 acarsd-info.nse
-rw-r--r-- 1 root root  8749 Sep 29 02:24 address-info.nse
-rw-r--r-- 1 root root  3345 Sep 29 02:24 afp-brute.nse
-rw-r--r-- 1 root root  6463 Sep 29 02:24 afp-ls.nse
-rw-r--r-- 1 root root  7001 Sep 29 02:24 afp-path-vuln.nse
-rw-r--r-- 1 root root  5600 Sep 29 02:24 afp-serverinfo.nse
-rw-r--r-- 1 root root  2621 Sep 29 02:24 afp-showmount.nse
-rw-r--r-- 1 root root  2262 Sep 29 02:24 ajp-auth.nse
-rw-r--r-- 1 root root  2983 Sep 29 02:24 ajp-brute.nse
[...]
```

The scripts fall into 14 categories:
| Category | Description |
|---|---|
| **auth** | Determination of authentication credentials. |
| **broadcast** | Scripts which are used for host discovery by broadcasting; the discovered hosts can be automatically added to the remaining scans. |
| **brute** | Executes scripts that try to log in to the respective service by brute-forcing with credentials. |
| **default** | Default scripts executed by using the `-sC` option. |
| **discovery** | Evaluation of accessible services. |
| **dos** | These scripts are used to check services for denial of service vulnerabilities and are used less as they harm the services. |
| **exploit** | This category of scripts tries to exploit known vulnerabilities for the scanned port. |
| **external** | Scripts that use external services for further processing. |
| **fuzzer** | Uses scripts to identify vulnerabilities and unexpected packet handling by sending different fields; this can take much time. |
| **intrusive** | Intrusive scripts that could negatively affect the target system. |
| **malware** | Checks if some malware infects the target system. |
| **safe** | Defensive scripts that do not perform intrusive or destructive actions. |
| **version** | Extension for service detection. |
| **vuln** | Identification of specific vulnerabilities. |

To specify specific scripts or categories of scripts to be run on a specific port, use the `--script` flag. To run multiple scripts or categories, separate them by a comma.
```bash
nmap --script <script>,<script> -p <port> <hosts>
```

To automatically let Nmap run a set of default scripts on open ports, use the `-sC` flag.
```bash
nmap -sC <hosts>
```

Sample script scan output:
```shell-session
$ nmap -sC 10.10.122.21
Starting Nmap 7.98 ( https://nmap.org ) at 2025-10-27 22:51 -0500
Nmap scan report for 10.10.122.21
Host is up (0.13s latency).
Not shown: 989 closed tcp ports (conn-refused)
PORT     STATE    SERVICE
22/tcp   open     ssh
| ssh-hostkey:
|   256 47:21:73:e2:6b:96:cd:f9:13:11:af:40:c8:4d:d6:7f (ECDSA)
|_  256 2b:5e:ba:f3:72:d3:b3:09:df:25:41:29:09:f4:7b:f5 (ED25519)
53/tcp   open     domain
| dns-nsid:
|   NSID: pdns (70646e73)
|_  id.server: pdns
512/tcp  open     exec
513/tcp  open     login
514/tcp  open     shell
873/tcp  open     rsync
901/tcp  filtered samba-swat
1069/tcp filtered cognex-insight
3000/tcp open     ppp
3306/tcp filtered mysql
8081/tcp filtered blackice-icecap

Nmap done: 1 IP address (1 host up) scanned in 33.18 seconds
```
Commonly, the `-sC` option is often used alongside `-sV`. The two options can also combined with a single `-sVC` flag.
```bash
nmap -sVC <hosts>
```

## OS Enumeration
The `-O` option tells Nmap to detect the operating system of the host(s) being scanned based on the fingerprints gathered. This option **requires root privileges** to be ran, and the target should have at least one open port and one closed port that Nmap can detect.
```bash
sudo nmap -O <hosts>
```

To combine service enumeration, default script scanning, and OS detection, we can use the aggressive scan option (`-A`). This scan type **requires root privileges** and generates a lot of traffic.
```bash
sudo nmap -A <hosts>
```
