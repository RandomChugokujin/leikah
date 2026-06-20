---
title: ESC3
description: Request certificate on behalf of another user with a enrollment agent certificate
categories: [Active Directory, AD CS]
tags: [Active Direcotory, Privilege Escalation, Lateral Movement, AD CS]
weight: 2
---

One of the **Extended Key Usages (EKUs)** for certificates issued by AD CS is **Certificate Enrollment Agent**, which allows the holder of the certificate to request certificates for another user as if they are that user. To abuse this for privilege escalation, there needs to be at least two templates matching conditions below:

Condition 1: A template allows a low-privileged user to enroll in an enrollment agent.
- Enrollment rights granted to a user or group for which we have access to.
- Manager approval is disabled.
- No authorized signatures are required.
- Certificate Enrollment Agent or Any Purpose is set as the EKU.

Condition 2: A template permit a low privileged user to use the enrollment agent certificate to request a certificate on behalf of another user that can be used for authentication.
- Enrollment rights granted to a user or group for which we have access to (including the user we can request a certificate for via condition 1).
- Manager approval is disabled.
- No authorized signatures are required.
- Client Authentication or Any Purpose is set as the EKU.

The chain of attack goes as the following:
1. Request a condition 1 certificate as the current user.
2. Use the condition 1 certificate to request a condition 2 certificate on behalf of the target user, which allows for client authentication.
3. Authenticate as the target user using condition 2 certificate.

## Linux Perspective
First, we request a certificate with **Certificate Enrollment Agent** listed as one of its EKUs, as our current controlled user.
```bash
certipy req -k -no-pass -ca '<ca_name>' -template "<agent_template>" -target "<dc_host>" \
-out controlled [-key-size 4096]
```

Next, request a certificate on behalf of a target user (specified with `-on-behalf-of`), while passing the certificate we received earlier back to the CA with `-pfx` to prove that we have rights as an enrollment agent. If we are targetting a user account, we can request a template from the built-in `User` template, which has **Client Authentication** listed under one of its EKUs.
{{% alert title="Note" %}}
The domain part of the user specified under the `-on-behalf-of` argument must not be FQDN (`CONTOSO` rather than `CONTOSO.COM`), or else the AD CS may reject your request.
{{% /alert %}}
```bash
certipy req -k -no-pass -ca '<ca_name>' -template user -target '<dc_host>' \
-on-behalf-of '<domain>\<target_user>' -pfx controlled.pfx -sid '<target_sid>' [-key-size 4096]
```

If we receive the certificate issued for the target user, we may Pass-the-Cert to obtain the user's TGT and NT hash.
```bash
certipy auth -pfx administrator.pfx -dc-ip '<dc_ip>'
```
