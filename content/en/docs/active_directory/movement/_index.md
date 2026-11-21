---
title: Lateral Movement and Privilege Escalation
description: Move from account to account, service to service, and machine to machine while escalating your privileges until you compromise the domain.
categories: [Active Directory]
tags: [Active Directory, Privilege Escalation, Lateral Movement]
weight: 2
---
Lateral movement and privilege escalation within an Active Directory domain are a gradual and cyclical process of analyzing the access our accounts have to domain resources and principals (other users, groups, machines, and services), and abusing that access to reach other machines, services, or more privileged accounts. We repeat this process with the end goal of achieving total domain compromise.

Misconfigurations of access rights and privileges enable an attacker's movement within Active Directory. Rather than obtaining access after exploiting a vulnerability, more commonly, the attacker simply **logs in**.
