---
title: SMB Relay Attack
description: SMB Relay Attack
categories: [Active Directory]
tags: [Active Direcotory, Initial Access, SMB, NTLM Authentication]
weight: 2
---

{{% alert title="Note" %}}
This article is under construction. Information presented is not complete.
{{% /alert %}}

SMB still supports NTLM authentication. The authentication flow goes as follows:
1. Client calculates NTLM hash from the user's password and sends the username to the server.
2. Server returns a random number called *nounce* as a **challenge**.
3. Client completes the challenge by encrypting the nounce using the NTLM hash and sending the **response** to the server.
4. If not part of an AD domain, the server encrypts the nounce itself and compare it to the ciphertext supplied by the client. If part of the AD domain, the server sends the client response to the **Domain Controller**, who does the comparison and tells the server if the response match or not.
5. If there is a match, the client is successfully authenticated.

This authentication follow is suspetible to a **Man-in-the-Middle** attack called SMB relay. The flow of the attack goes as follows:
1. Client initates connection to an **attacker controlled relay**.
2. Attacker relay connects to target server, relay client's username to target
3. Server responds the attacker relay with **NTLM challenge**.
4. Attacker relays the **NTLM challenge** to the client.
5. Client completes the challenges, sends attacker relay the **NTLM response**.
6. Attacker relays client's **NTLM response** to the target server.
7. Target server checks the response. If it's correct, access is granted to attacker relay.

TODO: Create Dedicated article for SMB relay attack
