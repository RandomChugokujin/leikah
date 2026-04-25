---
title: Lateral Movement and Privilege Escalation
description: Move from account to account, service to service, and machine to machine while escalating your privileges until you compromise the domain.
categories: [Active Directory]
tags: [Active Direcotory, Privilege Escalation, Lateral Movement]
weight: 2
---
Lateral movement and privilege escalation within the Active Directory domain is a gradual and cyclical process of analyzing the access our account(s) have on the domain's principals (other users, groups, machines, services), and abusing those accesses to either access other machines or services, or other more-privileged accounts. We repeat this process with the end-goal of achieving total domain compromise.

Misconfigurations of accesses and privileges are what enables attacker's movement with Active Directory. Rather than obtaining access after exploiting a vulnerability, more commonly, the attacker simply **logs in**.
