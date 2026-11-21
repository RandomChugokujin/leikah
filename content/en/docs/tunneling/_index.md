---
title: Port Forwarding & Tunneling
description: Port Forwarding, Tunneling, and Pivoting techniques to move laterally to an internal network.
categories: [Tunneling]
tags: [Tunneling]
weight: 2
---

Many enterprise networks are separated into the **Demilitarized Zone (DMZ)** and one or more internal networks. The DMZ is often used to host public-facing services, such as the organization's website, VPN servers, etc., while internal networks are what office workstations and internal servers are connected to. This separation helps minimize the impact of the compromise of one or more public-facing services spreading into the internal network.

However, there are also machines that serve as **jump hosts** to manage hosts on other networks. If such a host is compromised in the DMZ, it can allow an attacker to pivot into internal networks. Techniques such as SSH/socat port forwarding and SOCKS tunneling may be used to achieve this.
