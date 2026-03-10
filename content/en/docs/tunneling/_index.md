---
title: Port Forwarding & Tunneling
description: Port Fowarding, Tunneling, and Pivoting techniques to move latterally to an internal network.
categories: [Tunneling]
tags: [Tunneling]
weight: 2
---

Many enterprise networks are separated into the **Demilitarized Zone (DMZ)** and one or more internal networks. The DMZ is often used to host public-facing services, such as the organization's website, VPN servers, and etc., while internal networks are what office workstations and internal servers are connected to. This separation helps minimize the impact of the compromise of one or more public-facing service from spreading into the internal network.

However, there are also machines that serve as **jump hosts** that can be used to manage hosts on other networks. If such host is compromised in the DMZ, it can facilitate the attack to pivot into the internal networks. Techniques such as SSH/Socat Port Forwarding, SOCKS Tunneling and others may be used to achieve this end.
