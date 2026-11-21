---
title: Kerberos
description: Abusing the ticket-based authentication and authorization protocol that governs the operation of Active Directory
categories: [Active Directory]
tags: [Active Directory, Privilege Escalation, Lateral Movement, Kerberos]
weight: 2
---

Kerberos is a ticket-based network authentication protocol that also carries authorization data. This is the usual password-based process a client goes through to access a service in an Active Directory Kerberos network:
1. Client requests a **Ticket Granting Ticket (TGT)** from the **Authentication Service (AS)** of the **Key Distribution Center (KDC)** (AS-REQ), normally providing pre-authentication data.
2. The KDC authenticates the client, then sends back a response (AS-REP).
3. Client decrypts the encrypted reply data using a password-derived key, obtaining the TGT session key. The TGT itself is encrypted with the `krbtgt` account's key and is opaque to the client.
4. Client sends the TGT, an authenticator protected by the TGT session key, and the target **service principal name (SPN)** to the **Ticket Granting Service (TGS)** (TGS-REQ).
5. TGS verifies the request and applicable KDC policy, then returns a service ticket and encrypted reply data (TGS-REP). Issuing a ticket does not guarantee authorization to the service.
6. Client decrypts the reply data using the TGT session key to obtain the client/service session key. The service ticket is encrypted with the service account's long-term key.
7. Client presents the service ticket and a new authenticator to the service (AP-REQ).
8. The service decrypts the ticket with its account key, verifies the authenticator and relevant authorization data, and makes an access decision. PAC validation may require contacting a DC.

![Kerberos Authentication Flow in Active Directory Network](/img/kerberos/kerberos_auth_flow.jpg)

Microsoft's implementation of Kerberos sits at the center of Active Directory. **Domain Controllers (DCs)** act as KDCs, using Active Directory's account and authorization data. At the same time, different steps within the Kerberos authentication flow can be leveraged by attackers to obtain access to accounts and services. Attackers can use responses from the KDC to attempt offline password cracking (roasting attacks), extract Kerberos tickets from compromised machines, or forge tickets where they possess the required keys and validation permits it.

## References
Image Source: [link](https://www.manageengine.com/products/active-directory-audit/kb/images/event-4771-kerberos-authentication-illustration.jpg)
