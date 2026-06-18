---
title: AD CS
description: Abuse Active Directory Certificate Service to achieve lateral movement and total domain compromise.
categories: [Active Directory, AD CS]
tags: [Active Direcotory, Privilege Escalation, Lateral Movement, AD CS]
weight: 2
---
**Active Directory Certificate Services (AD CS)** is a Windows Server role for issuing and managing **public key infrastructure (PKI)** certificates used in secure communication and authentication protocols. Certificate configurations are pre-defined in **certificate templates**, which includes fields such as:
- SubjectAlternativeName (SAN): Defines one or more alternative names that the subjects may go by.
- Extended Key Usages (EKUs): Describe how the certificate will be used. Common ones include:
    - Client Authentication: for authenticating
    - Smart Card Logon: used for smart card authentication
    - Server Authentication: used for identifying server (e.g., HTTPS certificates)
    - Certificate Request Agent: enables a principal to request a certificate on behalf of another

The enrollment process for a client in AD CS goes as follows:
1. Client generates public-private key pair.
2. Client sends a certificate request along with its public key, the certificate template it wants, and various other settings.
3. CA checks if the certificate template exist, if the client is allowed to enroll in it, and if the settings in the request is allowed by the template.
4. CA generates a certificate and signs it with its private key.
5. Client stores the certificate and use it for the purpose outlined in the EKUs

Misconfiguration of CAs and certificate templates can lead to privilege escalation and even total domain compromise.

## Enumerating AD CS
Enumeration of AD CS can be done from Linux systems using [certipy](https://github.com/ly4k/Certipy) by ly4k.
```bash
# With username/password:
certipy find -u '<username>' -p '<password>' -dc-ip '<dc_ip>' [-ldap-scheme ldap]
# With Kerberos Ticket (ccache path exported in KRB5CCNAME environment variable)
certipy find -k -no-pass -target '<dc_host>' -ns '<dc_ip>'
```
Output in JSON and plain text will be generated, containing information regarding the configuration of the CAs and certificate templates available. Conveniently, `certipy` also highlights any potential vulnerabilities identified in those configurations, including ESC1-ESC16 privilege escalation vulnerabilities.

Alternatively, this can also be done on Windows systems using [certify](https://github.com/GhostPack/Certify).
```cmd
# Find vulnerable/abusable certificate templates using default low-privileged group
Certify.exe find /vulnerable

# Find vulnerable/abusable certificate templates using all groups the current user context is a part of:
Certify.exe find /vulnerable /currentuser
```
## Reference and Further Reading
[*Certified Pre-Owned: Abusing Active Directory Certificate Services*](https://specterops.io/wp-content/uploads/sites/3/2022/06/Certified_Pre-Owned.pdf) by Will Schroeder and Lee Christensen from SpecterOps.
