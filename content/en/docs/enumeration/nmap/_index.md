---
title: Nmap
description: Discover open ports and available services on your targets with Nmap
categories: [Enumeration, Network Enumeration]
tags: [Enumeration, Nmap]
weight: 2
---

Nmap is **the** go-to port scanner for security professionals and researchers for many years. It allows open ports on computers to be discovered over the network by sending packets to each port and analyze how the host responds.

Penetration Testers often use port scanners like Nmap to conduct **Active Recon** on the targets being assessed.

## TL;DR
Here are a few commands to get you started with nmap quickly:

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

[Ippsec](https://www.youtube.com/@ippsec)'s Nmap scan command as seen in his HTB walkthroughts:
- `-vv`: Double verbose output
- `-oA nmap/<filename_prefix>`: Save output in all three formats (normal, greppable, XML) to a directory
```bash
sudo nmap -sC -sV -vv -oA nmap/<filename_prefix> <hosts>
```

## References for This Section
- Nmap `--help` and manpage
- [Nmap online project guide](https://nmap.org/book/toc.html)
