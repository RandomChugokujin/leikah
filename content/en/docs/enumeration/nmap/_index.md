---
title: Nmap
description: Discover open ports and available services on your targets with Nmap
categories: [Enumeration, Network Enumeration]
tags: [Enumeration, Nmap]
weight: 2
---

Nmap has been **the** go-to port scanner for security professionals and researchers for many years. It discovers open ports on computers over the network by sending probes to selected ports and analyzing how the host responds.

Penetration testers often use port scanners like Nmap to conduct **Active Recon** on the targets being assessed.

## TL;DR
Here are a few commands to get you started with Nmap quickly:

Basic run:
```bash
nmap <hosts>
```

My favorite Nmap scan command for CTFs and exams:
- `-sVC`: Service enumeration + `default` NSE scripts
- `-T4`: Timing template 4, a relatively fast scanning pace
- `-oN <filename>`: Save output in normal plaintext
```bash
sudo nmap -sVC -T4 -oN <filename> <hosts>
```

[Ippsec](https://www.youtube.com/@ippsec)'s Nmap scan command as seen in his HTB walkthroughs:
- `-vv`: Double verbose output
- `-oA nmap/<filename_prefix>`: Save output in all three formats (normal, grepable, XML) to the `nmap` directory, which must already exist
```bash
sudo nmap -sC -sV -vv -oA nmap/<filename_prefix> <hosts>
```

## References for This Section
- Nmap `--help` and manpage
- [Nmap online project guide](https://nmap.org/book/toc.html)
