---
title: Kerberos
description: Abusing the ticket-based authentication and authorization protocol that governs the operation of Active Directory
categories: [Active Directory]
tags: [Active Direcotory, Privilege Escalation, Lateral Movement, Kerberos]
weight: 2
---

Kerberos is a ticket-based network protocol that enables centralized authentication and authorization management in a network. This is the process a client goes through to access a service in a Kerberos network:
1. Client requests a **Ticket Granting Ticket (TGT)**  from the **Auethenitcation Service (AS)** of the **Key Distribution Center (KDC)** (AS-REQ).
2. The KDC authenticates the client, then sends back a response (AS-REP).
3. Client decrypts the AS-REP using the hash of their password, obtaining the TGT.
4. Client hands the TGT to the **Ticket Granting Service (TGS)** alongside the **service principal name (SPN)** of the service they are attempting to access (TGS-REQ).
5. TGS after verifying the TGT and ensure the client can access the SPN, then responds to the client's request (TGS-REP).
6. Client decrypts TGS-REP, obtaining the service ticket.
7. Client hands the service ticket to the service.
8. The service decrypts the service ticket using the password hash of its service account and verifies its content, then grants the client access.

Microsoft's implementation of Kerberos sits at the center of Active Directory. The **Domain Controllers (DC)** acts as the KDC, enabling both centralized storage of credentials as well as user privileges and permissions. At the same time, different steps within the Kerberos authentication flow can be leveraged by attackers to obtain access to accounts and services. Attackers can use responses from the KDC to crack the target's password offline (roasting attacks), extract Kerberos tickets from compromised machines, or forge tickets to escalate their privileges.
