---
title: Nmap Scan Types
description: Nmap's scan methods and their pros and cons
categories: [Enumeration, Network Enumeration]
tags: [Enumeration, Nmap, Port Scanning]
weight: 2
---
Nmap offers a variety of port scan methods, each with its own pros and cons. Some types may see odd at first, but they often shine at specific use cases.

## TCP Connection Scan
By default, nmap uses **TCP Connection Scan** when ran **without root privileges**, which establishes:
- The port as **open** if the host completes the **TCP three-way handshake**.
- The port as **closed** if the host **resets** the attempt to connect.
- The port as **filtered** if the host **rejects** or **does not respond** to the attempt to connect

TCP connection scan can be manually specified using the `-sT` flag.
```bash
nmap -sT <hosts>
```
**Pros**: Highly Accurate

**Cons**: Noisy, Slow

## TCP SYN Scan
Instead of completing a three-way handshake like the TCP Connection Scan, the SYN Scan **resets** the three-way handshake when it receives the **SYN-ACK** packet from the host, and concludes that port as **open**. This is the default scan type of Nmap when ran **with root privileges**.

TCP SYN scan can be manually specified with the `-sS` flag. Note this scan type require privileged access to raw sockets since it needs to manually reset the TCP three-way handshake.
```bash
sudo nmap -sS <hosts>
```
**Pros**: Fast, Stealthy

**Cons**: Less accurate, Can still be detected by advanced IDS/IPS systems

Despite its shortcomings, the SYN Scan is the most popular Nmap port scan type.

## UDP Scan
Nmap also supports discovering services running on UDP ports. It marks the port as:
- **open** if Nmap gets a configured application response.
- **closed** if Nmap gets an **ICMP Type 3 Error 3 (Host Unreachable)** response.
- **open|filtered** if Nmap gets other ICMP responses or times out

Use the `-sU` flag for a UDP scan. Note this scan type **requires root privileges**.
```bash
sudo nmap -sU <hosts>
```
Note this scan type can take quite a long time due to UDP being a stateless protocol and the need for long timeouts to account for packet loss.

## TCP ACK Scan
The TCP ACK is not commonly used, but is nonetheless valuable as it helps to enumerate firewall rules on a host while evading IDS/IPS systems. It sends an **TCP ACK packet** instead of initiating a three-way handshake. If the the port is **unfiltered**, the host would reset the connection in response, allowing Nmap to conclude that connections to a particular port is not obstructed by firewall rules. This makes it harder for simple firewalls to block.

Use the `-sA` flag for a TCP ACK scan. This scan type also **requires root privileges**
```bash
sudo nmap -sA <hosts>
```
