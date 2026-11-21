---
title: ESC4
description: Leverage vulnerable certificate access control to escalate privileges.
categories: [Active Directory, AD CS]
tags: [Active Directory, Privilege Escalation, Lateral Movement, AD CS]
weight: 2
---

If a principal controlled by the attacker has sufficient rights over a certificate template, the attacker may reconfigure it into an ESC1 or ESC2 condition. **FullControl** permits direct changes; **WriteDacl** can grant the required access, and **WriteOwner** may allow taking ownership and then changing the DACL. **WriteProperty** must cover the relevant attributes, not merely an unrelated property. ESC1 involves enrollee-supplied identity information and an authentication-capable certificate; ESC2 involves **Any Purpose** or no EKUs and may require an enrollment-agent chain.

The template must already be published by a CA that permits enrollment; editing a template alone does not publish it. Effective enrollment rights, issuance requirements, and modern strong certificate mapping still apply, as described under [ESC1 and ESC2]({{< relref "/docs/active_directory/movement/adcs/esc1_esc2/">}}).

## Linux Perspective
`Certipy` 5.x can automatically configure a template to an ESC1-like state if the user has sufficient rights. This can alter its DACL and multiple issuance settings, affecting other users: retain the saved original configuration and restore it after the authorized test. The following uses the requester ccache exported in `KRB5CCNAME`.
```bash
certipy template -k -no-pass -template '<vuln_template>' -target '<dc_host>' -write-default-configuration
```

Next, follow the ESC1 request and authentication steps. Allow for CA template caching and AD replication, and verify the issued identity and SID rather than assuming that a supplied UPN guarantees impersonation.
```bash
certipy req -k -no-pass -ca '<ca_name>' -upn '<target_user>@<domain>' -sid '<target_user_sid>' -template '<vuln_template>' -target '<ca_host>' -out target
```
```bash
certipy auth -pfx target.pfx -dc-ip '<dc_ip>'
```
