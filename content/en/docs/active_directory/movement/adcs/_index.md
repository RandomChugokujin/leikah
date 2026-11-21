---
title: AD CS
description: Abuse Active Directory Certificate Services to achieve lateral movement and potentially domain compromise.
categories: [Active Directory, AD CS]
tags: [Active Directory, Privilege Escalation, Lateral Movement, AD CS]
weight: 2
---
**Active Directory Certificate Services (AD CS)** is a Windows Server role for issuing and managing **public key infrastructure (PKI)** certificates used in secure communication and authentication protocols. Enterprise CAs use **certificate templates** to define certificate configurations, including fields such as:
- SubjectAlternativeName (SAN): Defines one or more alternative names that the subjects may go by.
- Extended Key Usages (EKUs): Describe how the certificate will be used. Common ones include:
    - Client Authentication: for authenticating
    - Smart Card Logon: used for smart card authentication
    - Server Authentication: used for identifying a server (e.g., HTTPS certificates)
    - Certificate Request Agent: enables a principal to request a certificate on behalf of another

The enrollment process for a client in AD CS goes as follows:
1. Client generates a public-private key pair.
2. Client sends a certificate request along with its public key, the certificate template it wants, and various other settings.
3. CA checks whether the certificate template is published on that CA, whether the client is allowed to enroll, and whether the request meets the template's issuance requirements.
4. CA generates a certificate and signs it with its private key.
5. Client stores the certificate alongside its private key and uses it for a purpose permitted by the certificate and the relying service.

Misconfiguration of CAs and certificate templates can lead to privilege escalation and even total domain compromise.

## Enumerating AD CS
Enumeration of AD CS can be done from Linux systems using [certipy](https://github.com/ly4k/Certipy) by ly4k.
```bash
# With username/password:
certipy find -u '<username>' -p '<password>' -dc-ip '<dc_ip>' [-ldap-scheme ldap]
# With Kerberos Ticket (ccache path exported in KRB5CCNAME environment variable)
certipy find -k -no-pass -target '<dc_host>' -ns '<dc_ip>'
```
Output in JSON and plain text will be generated, containing information regarding the configuration of the CAs and certificate templates available. Conveniently, `certipy` also highlights potential ESC misconfigurations; coverage depends on the installed version. A finding is not proof of exploitability: CA publication, effective enrollment rights, issuance requirements, certificate mapping, and patch levels still matter.

Alternatively, this can also be done on Windows systems using [Certify](https://github.com/GhostPack/Certify). The commands below use the legacy Certify 1.x syntax; Certify 2.x has a different command-line interface.
```cmd
# Find vulnerable/abusable certificate templates using default low-privileged group
Certify.exe find /vulnerable

# Find vulnerable/abusable certificate templates using all groups the current user context is a part of:
Certify.exe find /vulnerable /currentuser
```
## Reference and Further Reading
[*Certified Pre-Owned: Abusing Active Directory Certificate Services*](https://specterops.io/wp-content/uploads/sites/3/2022/06/Certified_Pre-Owned.pdf) by Will Schroeder and Lee Christensen from SpecterOps.
