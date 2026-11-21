---
title: Nmap Scan Types
description: Nmap's scan methods and their pros and cons
categories: [Enumeration, Network Enumeration]
tags: [Enumeration, Nmap, Port Scanning]
weight: 2
---
Nmap offers a variety of port scan methods, each with its own pros and cons. Some types may seem odd at first, but they often shine in specific use cases.

## TCP Connection Scan
By default, Nmap uses **TCP Connection Scan** when run **without raw-packet privileges on Unix**, which establishes:
- The port as **open** if the host completes the **TCP three-way handshake**.
- The port as **closed** if the host **resets** the attempt to connect.
- The port as **filtered** if the connection attempt times out or receives a filtering-related error (a TCP reset instead means **closed**).

TCP connection scan can be manually specified using the `-sT` flag.
```bash
nmap -sT <hosts>
```
**Pros**: Highly Accurate

**Cons**: Noisy, Slow

## TCP SYN Scan
Instead of completing a three-way handshake like the TCP Connection Scan, the SYN Scan sends a **reset** when it receives the **SYN-ACK** packet from the host, and concludes that the port is **open**. This is Nmap's default TCP scan when raw-packet privileges are available.

TCP SYN scan can be manually specified with the `-sS` flag. On Unix, this requires root or equivalent raw-packet privileges to craft and capture packets.
```bash
sudo nmap -sS <hosts>
```
**Pros**: Fast, Reliable, Fewer completed connections than a connect scan

**Cons**: Requires raw-packet privileges, Can still be detected by firewalls and IDS/IPS systems

Despite its shortcomings, the SYN Scan is the most popular Nmap port scan type.

## UDP Scan
Nmap also supports discovering services running on UDP ports. It marks the port as:
- **open** if Nmap gets a UDP response.
- **closed** if Nmap gets an **ICMP Type 3, Code 3 (Port Unreachable)** response (IPv4).
- **filtered** if Nmap gets certain other ICMP unreachable responses (IPv4 Type 3, Codes 0, 1, 2, 9, 10, or 13).
- **open|filtered** if Nmap gets no response after retransmissions.

Use the `-sU` flag for a UDP scan. Note this scan type **requires raw-packet privileges**, normally root on Unix.
```bash
sudo nmap -sU <hosts>
```
This scan type can take quite a long time because open/filtered ports may not reply, requiring timeouts and retransmissions, and hosts often rate-limit ICMP Port Unreachable responses.

## TCP ACK Scan
The TCP ACK scan helps map firewall rules. It sends a **TCP ACK packet** instead of initiating a three-way handshake. A reset means the port is **unfiltered** for the ACK probe; no response or certain ICMP unreachable errors mean **filtered**. It does not distinguish open ports from closed ports, prove that new connections are allowed, or guarantee IDS/IPS evasion.

Use the `-sA` flag for a TCP ACK scan. This scan type also **requires raw-packet privileges**, normally root on Unix.
```bash
sudo nmap -sA <hosts>
```
