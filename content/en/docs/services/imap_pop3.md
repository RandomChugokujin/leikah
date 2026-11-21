---
title: "IMAP/POP3"
description: Email retrieval protocols
categories: [Services]
tags: [Services, IMAP, POP3, Email]
---

## Service Info
- Name: Internet Message Access Protocol (IMAP) / Post Office Protocol 3 (POP3)
- Purpose: Retrieving and managing email on a remote mail server.
- Listening port:
    - POP3: 110/TCP, 995/TCP (implicit TLS)
    - IMAP: 143/TCP, 993/TCP (implicit TLS)
- OS: Unix-Like, Windows

**Internet Message Access Protocol (IMAP)** allows management of emails directly on remote servers. It synchronizes local email clients with mailboxes on the server while organizing them into folder-like structures. IMAP is unencrypted by default, and TLS may be used to establish an encrypted IMAP session.

**Post Office Protocol 3 (POP3)** is a more minimal counterpart to IMAP, primarily used to download mail from a remote server. Downloading a message with `RETR` does not itself delete it: clients normally mark messages with `DELE`, and deletion is committed when the session ends with `QUIT`. Clients can leave messages on the server. Port 110 starts without encryption unless upgraded with `STLS`; port 995 uses implicit TLS.

## Service Commands
Like SMTP, both IMAP and POP3 use text commands.

### IMAP Commands
- `1 LOGIN username password`: User's login.
- `1 LIST "" *`: Lists all directories.
- `1 CREATE "ToRead"`: Creates a mailbox with the specified name.
- `1 DELETE "ToRead"`: Deletes a mailbox; deleting `INBOX` is not permitted.
- `1 RENAME "ToRead" "Important"`: Renames a mailbox.
- `1 LSUB "" *`: Returns a subset of names from the set of names that the user has declared as being active or subscribed.
- `1 SELECT INBOX`: Selects a mailbox so that messages in the mailbox can be accessed.
- `1 UNSELECT`: Exits the selected mailbox without expunging messages (requires UNSELECT support on IMAP4rev1 servers).
- `1 FETCH <ID> ALL`: Retrieves flags, internal date, size, and envelope for a message sequence number; use `1 FETCH <ID> BODY[]` to retrieve its full contents.
- `1 CLOSE`: Closes the selected mailbox and permanently removes messages marked `\Deleted` if it is open read-write.
- `1 LOGOUT`: Closes the connection with the IMAP server.

### POP3 Commands
- `USER username`: Identifies the user.
- `PASS password`: Authenticates the user using their password.
- `STAT`: Requests the message count and total size in octets.
- `LIST`: Lists message numbers and sizes in octets.
- `RETR id`: Requests the server to deliver the requested email by ID.
- `DELE id`: Marks the message number for deletion when `QUIT` ends the transaction.
- `CAPA`: Requests the server to display the server capabilities.
- `RSET`: Clears deletion marks set during the session.
- `QUIT`: Ends the session and commits pending deletions if issued from the transaction state.

## Service Scanning
Nmap Scan:
```bash
sudo nmap <host> -sVC -p110,143,993,995
```
A candidate hostname may be found in the certificate's `commonName` or subject alternative names.
```txt
Starting Nmap 7.80 ( https://nmap.org ) at 2021-09-19 22:09 CEST
Nmap scan report for 10.129.14.128
Host is up (0.00026s latency).

PORT    STATE SERVICE  VERSION
110/tcp open  pop3     Dovecot pop3d
|_pop3-capabilities: AUTH-RESP-CODE SASL STLS TOP UIDL RESP-CODES CAPA PIPELINING
| ssl-cert: Subject: commonName=mail1.gundam.local/organizationName=Inlanefreight/stateOrProvinceName=California/countryName=US
| Not valid before: 2021-09-19T19:44:58
|_Not valid after:  2295-07-04T19:44:58
143/tcp open  imap     Dovecot imapd
|_imap-capabilities: more have post-login STARTTLS Pre-login capabilities LITERAL+ LOGIN-REFERRALS OK LOGINDISABLEDA0001 SASL-IR ENABLE listed IDLE ID IMAP4rev1
| ssl-cert: Subject: commonName=mail1.gundam.local/organizationName=Inlanefreight/stateOrProvinceName=California/countryName=US
| Not valid before: 2021-09-19T19:44:58
|_Not valid after:  2295-07-04T19:44:58
993/tcp open  ssl/imap Dovecot imapd
|_imap-capabilities: more have post-login OK capabilities LITERAL+ LOGIN-REFERRALS Pre-login AUTH=PLAINA0001 SASL-IR ENABLE listed IDLE ID IMAP4rev1
| ssl-cert: Subject: commonName=mail1.gundam.local/organizationName=Inlanefreight/stateOrProvinceName=California/countryName=US
| Not valid before: 2021-09-19T19:44:58
|_Not valid after:  2295-07-04T19:44:58
995/tcp open  ssl/pop3 Dovecot pop3d
|_pop3-capabilities: AUTH-RESP-CODE USER SASL(PLAIN) TOP UIDL RESP-CODES CAPA PIPELINING
| ssl-cert: Subject: commonName=mail1.gundam.local/organizationName=Inlanefreight/stateOrProvinceName=California/countryName=US
| Not valid before: 2021-09-19T19:44:58
|_Not valid after:  2295-07-04T19:44:58
MAC Address: 00:00:00:00:00:00 (VMware)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 12.74 seconds
```

## Service Interaction
We can use `telnet` to interact with the unencrypted IMAP and POP3 services, allowing us to communicate with both services using their respective commands.
```bash
telnet <host> 110
telnet <host> 143
```

The `openssl` utility may also be used to communicate with IMAP and POP3 over implicit TLS.
```bash
openssl s_client -connect <host>:pop3s
openssl s_client -connect <host>:imaps
```

## Service Attacks

### Password Attacks
Hydra may be used to conduct password attacks against both IMAP(S) and POP3(S). On a mail server, it is very likely that these services are managed by the same software as SMTP. More often than not, using Hydra against one of the three services is enough to uncover user credentials for the email service.
```bash
hydra -l <username> -P <password_list> <host> <imap|imaps|pop3|pop3s>
```

### Reading Email
If we find a set of credentials, we can connect to the IMAP/POP3 services to read emails belonging to that user. In the captured IMAP example below, use `1 LIST "" *` (with a space before `*`); the recorded command omits that separator.

IMAP:
```txt
1 login robin Password123!
1 list ""*
* LIST (\Noselect \HasChildren) "." DEV
* LIST (\Noselect \HasChildren) "." DEV.DEPARTMENT
* LIST (\HasNoChildren) "." DEV.DEPARTMENT.INT
* LIST (\HasNoChildren) "." INBOX
1 select "DEV.DEPARTMENT.INT"
* FLAGS (\Answered \Flagged \Deleted \Seen \Draft)
* OK [PERMANENTFLAGS (\Answered \Flagged \Deleted \Seen \Draft \*)] Flags permitted.
* 1 EXISTS
* 0 RECENT
* OK [UIDVALIDITY 1636414279] UIDs valid
* OK [UIDNEXT 2] Predicted next UID
1 OK [READ-WRITE] Select completed (0.008 + 0.000 + 0.007 secs).
1 fetch 1 BODY[]
* 1 FETCH (BODY[] {167}
Subject: Flag
To: Robin <robin@gundam.local>
From: CTO <devadmin@gundam.local>
Date: Wed, 03 Nov 2021 16:13:27 +0200

Hi,

Hope you are having a good day.

Best,
CTO
)
1 OK Fetch completed (0.006 + 0.000 + 0.005 secs).
```

POP3:
```txt
USER marlin@gundam.local
+OK Send your password
PASS VeryStrongPassword2026!
+OK Mailbox locked and ready
LIST
+OK 1 messages (601 octets)
1 601
.
RETR 1
+OK 601 octets
Return-Path: marlin@gundam.local
Received: from [10.10.14.33] (Unknown [10.10.14.33])
        by WINSRV02 with ESMTPA
        ; Wed, 20 Apr 2022 14:49:32 -0500
Message-ID: <85cb72668d8f5f8436d36f085e0167ee78cf0638.camel@gundam.local>
Subject: Password change
From: marlin <marlin@gundam.local>
To: administrator@gundam.local
Cc: marlin@gundam.local
Date: Wed, 20 Apr 2022 15:49:11 -0400
Content-Type: text/plain; charset="UTF-8"
User-Agent: Evolution 3.38.3-1
MIME-Version: 1.0
Content-Transfer-Encoding: 7bit

Hi admin,

How can I change my password to something more secure?

Best,
Marlin
.
```
