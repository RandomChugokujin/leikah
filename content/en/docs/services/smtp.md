---
title: SMTP
description: Simple Mail Transfer Protocol
categories: [Services]
tags: [Services, SMTP, Email]
---

## Service Info
- Name: Simple Mail Transfer Protocol (SMTP)
- Purpose: Sending emails over an IP network.
- Listening ports: **25/TCP** (mail transfer), **587/TCP** (message submission, commonly upgraded with STARTTLS), **465/TCP** (submission over implicit TLS)
- OS: Unix-Like, Windows

SMTP facilitates the transfer of mail between a client and a mail server, or between two mail servers. Original SMTP included neither user authentication nor transport encryption. ESMTP provides an extension framework used by features such as `AUTH` and `STARTTLS`; support and policy must be checked on each server.

The process of sending an email using SMTP is as follows:
1. A **Mail User Agent (MUA)** submits a message, consisting of headers and a body, to a **Mail Submission Agent (MSA)**.
2. The submission server applies policy checks and queues the message for transfer.
3. One or more **Mail Transfer Agents (MTAs)** relay the message to the recipient's mail system.
4. A **Mail Delivery Agent (MDA)** delivers the message to the recipient's mailbox.

### SMTP Commands
SMTP communications are facilitated with commands. Common SMTP commands include:
- `AUTH PLAIN`: AUTH is a service extension used to authenticate the client.
- `HELO`: Identifies the SMTP client to the server; it does not authenticate the user.
- `EHLO`: Extended version of the `HELO` command. The server would respond with a list of its capabilities
- `MAIL FROM`: The client names the email sender.
- `RCPT TO`: The client names the email recipient.
- `DATA`: The client initiates the transmission of the email.
- `RSET`: The client aborts the initiated transmission but keeps the connection between client and server.
- `VRFY`: The client checks if a mailbox is available for message transfer.
- `EXPN`: Requests the membership of a mailing list, if the server permits expansion.
- `NOOP`: The client requests a response from the server to prevent disconnection due to time-out.
- `QUIT`: The client terminates the session.


## Service Enumeration

The default script scan (`-sC`) includes `smtp-commands`, which uses `EHLO` and `HELP` to discover supported extensions and commands.
```shell-session
╭─brian@rx-93-nu ~
╰─$ sudo nmap 10.10.0.25 -sC -sV -p25

Starting Nmap 7.80 ( https://nmap.org ) at 2021-09-27 17:56 CEST
Nmap scan report for 10.10.0.25
Host is up (0.00025s latency).

PORT   STATE SERVICE VERSION
25/tcp open  smtp    Postfix smtpd
|_smtp-commands: mail01.gundam.local, PIPELINING, SIZE 10240000, VRFY, ETRN, ENHANCEDSTATUSCODES, 8BITMIME, DSN, SMTPUTF8, CHUNKING,
MAC Address: 00:00:00:00:00:00 (VMware)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 14.09 seconds
```

## Service Interaction
Interaction with an SMTP server can be done using the `telnet` utility
```bash
telnet <host> 25
```
After connecting to the SMTP server, we may use the `EHLO` command to greet the server and get a list of available features.
```shell-session
╭─brian@rx-93-nu ~
╰─$ telnet 10.10.0.25 25

Trying 10.10.0.25...
Connected to 10.10.0.25.
Escape character is '^]'.
220 ESMTP Server


HELO mail01.gundam.local

250 mail01.gundam.local


EHLO mail1

250-mail01.gundam.local
250-PIPELINING
250-SIZE 10240000
250-ETRN
250-ENHANCEDSTATUSCODES
250-8BITMIME
250-DSN
250-SMTPUTF8
250 CHUNKING
```

### User Enumeration
Commands such as `VRFY`, `EXPN`, and `RCPT TO` may expose valid recipients, depending on server policy. A `252` reply means the server cannot verify the user but will accept a message for attempted delivery; it does not confirm that the account exists.

VRFY:
```shell-session
VRFY root

252 2.0.0 root


VRFY www-data

252 2.0.0 www-data


VRFY new-user

550 5.1.1 <new-user>: Recipient address rejected: User unknown in local recipient table
```

`EXPN` requests expansion of a distribution list when supported and permitted.
- `EXPN all` only expands a list named `all` if one exists and expansion is allowed; it is not a general command to list every user.

```shell-session
EXPN john

250 2.1.0 john@gundam.local


EXPN support-team

250 2.0.0 carol@gundam.local
250 2.1.5 elisa@gundam.local
```

`RCPT TO` identifies an envelope recipient and can be repeated for multiple recipients. Differences in responses may reveal valid recipients, but catch-all acceptance and deferred validation can obscure this. In the historical capture below, the stray `it is` line is not an SMTP command; use `MAIL FROM:<test@htb.com>` and `RCPT TO:<address>` syntax.
```shell-session
MAIL FROM:test@htb.com
it is
250 2.1.0 test@exmaple.com... Sender ok


RCPT TO:julio

550 5.1.1 julio... User unknown


RCPT TO:kate

550 5.1.1 kate... User unknown


RCPT TO:john

250 2.1.5 john... Recipient ok
```

The process of enumerating users may be automated using [smtp-user-enum](https://github.com/pentestmonkey/smtp-user-enum).
- Use `-M` to specify method (`VRFY`, `EXPN`, or `RCPT`).
- Use `-U` to specify wordlist.
- Use `-D` to specify domain.

```bash
smtp-user-enum -M <command> -U <userlist> -D <domain> -t <host>
```

### Sending Email
We can send an email to valid recipients within a `telnet` session with an SMTP server if its relay/submission policy permits it. The captured message below is missing the required blank line between headers and body; insert one after the `Date:` header when reproducing it. Also use `Sep`, not `Sept`, as the month abbreviation in that header.
```shell-session
EHLO gundam.local

250-mail01.gundam.local
250-PIPELINING
250-SIZE 10240000
250-ETRN
250-ENHANCEDSTATUSCODES
250-8BITMIME
250-DSN
250-SMTPUTF8
250 CHUNKING


MAIL FROM: <brian@gundam.local>

250 2.1.0 Ok


RCPT TO: <john@gundam.local> NOTIFY=success,failure

250 2.1.5 Ok


DATA

354 End data with <CR><LF>.<CR><LF>

From: <brian@gundam.local>
To: <john@gundam.local>
Subject: DB
Date: Tue, 28 Sept 2021 16:32:51 +0200
Good morning and I wish you a happy day!
.

250 2.0.0 Ok: queued as 6E1CF1681AB


QUIT

221 2.0.0 Bye
Connection closed by foreign host.
```

Alternatively, we can use [swaks](https://github.com/jetmore/swaks), a command-line SMTP testing tool, to send mail.
```bash
swaks --from <sender> --to <recipient> --header <email_header> --body <email_body> --server <host>
```

## Password Attacks
Hydra can be used to perform a password spray or brute-force against SMTP.
```bash
hydra -L <user_list> -p <password> -f <target> smtp
```
