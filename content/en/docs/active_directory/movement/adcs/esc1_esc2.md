---
title: ESC1 and ESC2
description: Abuse enrollee-supplied identities or overly broad certificate purposes
categories: [Active Directory, AD CS]
tags: [Active Directory, Privilege Escalation, Lateral Movement, AD CS]
weight: 2
---

ESC1 and ESC2 target different certificate template misconfigurations. ESC1 combines enrollee-supplied subject information with a certificate usable for authentication. The `ENROLLEE_SUPPLIES_SUBJECT` flag allows the requester to supply subject information, including an alternative identity in the Subject Alternative Name (SAN). If the issued certificate maps to the target account, **Pass-the-Certificate** can then obtain that account's Kerberos TGT.

The distinction is:
- ESC1:
    - An authentication-capable EKU, such as **Client Authentication**, is present.
    - `ENROLLEE_SUPPLIES_SUBJECT` flag is set.
- ESC2:
    - **Any Purpose** is set as an EKU, or no EKUs are specified.
    - `ENROLLEE_SUPPLIES_SUBJECT` is not required. Without control of the certificate identity, the certificate normally authenticates as the requester; escalation can instead require an enrollment-agent chain as described in [ESC3]({{< relref "/docs/active_directory/movement/adcs/esc3/">}}).

Both require effective enrollment rights on a template published by an accessible CA, with no unmet approval or signature requirements. PKINIT also requires a suitable DC certificate and a trusted issuing chain.

{{% alert title="Version caveat" %}}
On updated DCs, a supplied UPN alone is not sufficient: certificate authentication must satisfy strong mapping. KB5014754 made enforcement the default with the February 2025 updates and removed the Compatibility-mode override with the September 9, 2025 updates. With enrollee-supplied subjects, a CA can honor a requested SID security extension, so strong mapping does not eliminate ESC1. Certipy 5.x accepts `-sid`, but that option does not itself bypass CA issuance rules or a conflicting SID security extension. Check the identity and SID actually issued before assuming the certificate can impersonate the target.
{{% /alert %}}

## Linux Perspective
For ESC1, we use `certipy` 5.x to request a certificate from the vulnerable template, specifying the target identity with `-upn` and `-sid`. Here `-target` is the CA host, which need not be a DC; Kerberos authentication requires a valid requester ccache in `KRB5CCNAME`.
```bash
certipy req \
-k -no-pass \
-target '<ca_host>' -ns '<dc_ip>' \
-ca '<ca_name>' -template '<vuln_template>' \
-upn '<target_user>@<domain>' -sid '<target_user_sid>'
```

After obtaining a usable certificate, we use `certipy`, once again, to Pass-the-Certificate and obtain the mapped user's TGT. Certipy also attempts to retrieve the NT hash via U2U (UnPAC the hash).
```bash
certipy auth -pfx '<cert_path>' -dc-ip '<dc_ip>'
```

## Windows Perspective
From Windows systems, legacy Certify 1.x can request an alternative UPN with `/altname`. This UPN-only example is not sufficient on its own under strong mapping; Certify 2.x uses different options. The CA configuration string includes both the CA host and CA name:
```cmd
Certify.exe request /ca:"<ca_host>\<ca_name>" /template:"<vuln_template>" /altname:"<target_user>@<domain>"
```
Then, we can use `Rubeus` to Pass-the-Certificate and obtain a TGT. Supply a PFX containing the certificate and private key (a file or base64-encoded PFX), not a bare public certificate. Convert legacy Certify's PEM certificate and private key to PFX first.
```cmd
Rubeus.exe asktgt /user:"<target_user>" /certificate:"<pfx_path>" /password:"<cert_pass>" /domain:"<domain>" /dc:"<dc_host>" /show
```

