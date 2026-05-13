---
title: SMTP
description: Simple Mail Transfer Protocol
categories: [Services]
tags: [Services, SMTP, Email]
---

## Service Info
- Name: Simple Mail Transfer Protocol (SMTP)
- Purpose: Sending emails over an IP network.
- Listening port: **TCP port 25**, **TCP port 587** (Encrypted Transport)
- OS: Unix-Like, Windows

SMTP faciliates the transfer of mail between a client and a mail server, or between two mail servers. Originally, SMTP did not include user authencation nor transport encryption. Both features are implemented in **Extended Simple Mail Transfer Protocol (ESMTP)**, which faciliates most mail services today.

The process of sending an email using SMTP is as follows:
1. The SMTP client (**Mail User Agent**) converts email into a header and a body and uploads both to the SMTP Server (**Mail Transfer Agent**)
2. MTA checks email for size and spam then stores it.
3. MTA sends email to the destination SMTP Server (**Mail Delivery Agent**), where the data packets will be reassembled into a complete email.
4. Mail Delivery Agent transfers it to the recipient's mailbox

### SMTP Commands
SMTP communications are facilitated with commands. Common SMTP commands include:
- `AUTH PLAIN`: AUTH is a service extension used to authenticate the client.
- `HELO`: The client logs in with its computer name and thus starts the session.
- `EHLO`: Extended version of the `HELO` command. The server would respond with a list of its capabilities
- `MAIL FROM`: The client names the email sender.
- `RCPT TO`: The client names the email recipient.
- `DATA`: The client initiates the transmission of the email.
- `RSET`: The client aborts the initiated transmission but keeps the connection between client and server.
- `VRFY`: The client checks if a mailbox is available for message transfer.
- `EXPN`: The client also checks if a mailbox is available for messaging with this command.
- `NOOP`: The client requests a response from the server to prevent disconnection due to time-out.
- `QUIT`: The client terminates the session.


## Service Enumeration

The default script scan (`-sC`) runs `smtp-command`, which uses the `EHLO` command to list out the available commands on the server.
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
Commands such as `VRFY`, `EXPN`, and `RCPT TO` may be used to enumerate users on the system.

VRFY:
```shell-session
VRFY root

252 2.0.0 root


VRFY www-data

252 2.0.0 www-data


VRFY new-user

550 5.1.1 <new-user>: Recipient address rejected: User unknown in local recipient table
```

`EXPN` is similiar to `VRFY`, but when used with a distribution list, it will list all users on that list.
- A quick way to get all users on the system is to try `EXPN all`

```shell-session
EXPN john

250 2.1.0 john@gundam.local


EXPN support-team

250 2.0.0 carol@gundam.local
250 2.1.5 elisa@gundam.local
```

The `RCPT TO` is usually used to identify the recipient of the email, but it can be repeated multiple times for a given message to deliver a message to multiple recipients. We can leverage this to identify users.
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
We can send an email to a number of valid recipients within the `telnet` session with an SMTP server.
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

Alternatively, we can use [swaks](https://github.com/jetmore/swaks), a command line SMTP testing tool to send mail.
```bash
swaks --from <sender> --to <recipient> --header <email_header> --body <email_body> --server <host>
```

## Password Attacks
Hydra can be used to perform a password spray or brute-force against SMTP.
```bash
hydra -L <user_list> -p <password> -f <target> smtp
```
