---
title: FTP
description: File Transfer Protocol
categories: [Services]
tags: [Services, FTP]
weight: 2
---

## Service Info
- Name: File Transfer Protocol (FTP)
- Purpose: Transferring and sharing files over the network
- Listening port: **TCP port 21**
- OS: Unix-Like (more commonly), Windows

FTP has two channels of communication:
- **Control Connection**: used by the client to send commands and by the server to respond with status codes
- **Data Connection**: used for data transfer between the client and server

### Active vs. Passive Connections
FTP has two transfer modes, **active** and **passive**. The main difference is who initiates the data connection when a file is being transferred.
- **Active**: The client opens the control connection to server TCP port 21 and advertises a listening address/port using `PORT` or `EPRT`. The server initiates the data connection to that port (traditionally from TCP port 20); the client port need not be `N+1`.
- **Passive**: The client opens the control connection to server TCP port 21. `PASV` or `EPSV` requests a server listening port, and the client initiates the data connection to it. The interactive `passive` command is a client-side toggle, where supported.

The main reason for passive-mode FTP is that many clients, often desktops and workstations, have firewalls installed, which could block the server's data connection to the client during active mode. Firewalls tend to be less restrictive about outgoing connections. Therefore, in passive mode, the client initiates the data connection.

## Footprinting
Nmap service and default script scan:
```bash
sudo nmap -sV -p21 -sC -A <host>
```
Default NSE scripts for the FTP service include:
- `ftp-anon` checks if the FTP server allows anonymous access. If so, it lists the contents of the FTP root for the anonymous user
- `ftp-syst` sends `SYST` and `STAT` to obtain system and server-status information.

### Manual Banner Grabbing
Use Netcat for a plaintext TCP connection:
```bash
nc -nv <host> 21
```

Use `openssl` if TLS is enabled:
```bash
openssl s_client -connect <host>:21 -starttls ftp
```

### Anonymous Login
FTP has an option to allow anonymous users to log in to the server. To check if an FTP server has that option enabled, use the `ftp-anon` NSE script mentioned above, or try logging in via the `ftp` client.
```shell-session
$ ftp 10.129.14.136

Connected to 10.129.14.136.
220 (vsFTPd 3.0.5)
Name (10.129.14.136:brian): anonymous

230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls

200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
-rw-rw-r--    1 1002     1002      8138592 Sep 14 16:54 Calender.pptx
drwxrwxr-x    2 1002     1002         4096 Sep 14 16:50 Clients
drwxrwxr-x    2 1002     1002         4096 Sep 14 16:50 Documents
drwxrwxr-x    2 1002     1002         4096 Sep 14 16:50 Employees
-rw-rw-r--    1 1002     1002           41 Sep 14 16:45 Important Notes.txt
226 Directory send OK.
```

### FTP Client
The FTP client (`ftp`) can be used to browse the files and directories on the FTP server
```bash
ftp <host>
```

Below are a few basic FTP client commands. Support varies by client and server; listing flags such as `-a` and `-R` are not portable FTP protocol options.
- `ls <dir>`: list directory
- `ls -a <dir>`: list directory, including hidden files
- `ls -R <dir>`: recursively list a directory
- `cd <dir>`: change directory
- `get <file>`: download remote file
- `put <file>`: upload local file
- `help`: list available commands
- `! <cmd>`: execute command locally
- `passive`: Toggle active/passive mode
- `bye`/`quit`: disconnect from server and exit the client

### Netcat Manual Interaction
Alternatively, we can also manually interact with the service using Netcat. Use the `USER <username>` and `PASS <password>` commands to log in.
```shell-session
$ nc localhost 21
220 (vsFTPd 3.0.5)
USER anonymous
331 Please specify the password.
PASS pass
230 Login successful.
```
After logging in, we can use commands like `HELP`, `FEAT`, and `STAT` to further enumerate the service:
```shell-session
HELP
214-The following commands are recognized.
 ABOR ACCT ALLO APPE CDUP CWD  DELE EPRT EPSV FEAT HELP LIST MDTM MKD
 MODE NLST NOOP OPTS PASS PASV PORT PWD  QUIT REIN REST RETR RMD  RNFR
 RNTO SITE SIZE SMNT STAT STOR STOU STRU SYST TYPE USER XCUP XCWD XMKD
 XPWD XRMD
214 Help OK.
```
```shell-session
FEAT
211-Features:
 EPRT
 EPSV
 MDTM
 PASV
 REST STREAM
 SIZE
 TVFS
 UTF8
211 End
```
```shell-session
STAT
211-FTP server status:
     Connected to 127.0.0.1
     Logged in as ftp
     TYPE: ASCII
     No session bandwidth limit
     Session timeout in seconds is 300
     Control connection is plain text
     Data connections will be plain text
     At session startup, client count was 1
     vsFTPd 3.0.5 - secure, fast, stable
211 End of status
```

### Download All Available Files
We can use the following `wget` command to download all files accessible to us on an FTP share:
```bash
wget -m ftp://<username>:<password>@<host>
```

The `--no-passive-ftp` option disables passive transfer mode:
```bash
wget --no-passive-ftp -m ftp://<username>:<password>@<host>
```

If the username or password contains URL-special characters, use the `--user` and `--password` flags to specify the credentials separately. Quote or escape shell-special characters as well; these flags do not disable shell interpretation.
```bash
wget -m --user='<username>' --password='<password>' ftp://<host>
```

## References
[Stack Overflow: Downloading all files from an FTP Server](https://stackoverflow.com/questions/3003135/downloading-all-files-from-an-ftp-server)
[Hacktricks: Pentesting FTP](https://book.hacktricks.wiki/en/network-services-pentesting/pentesting-ftp/index.html#some-ftp-commands)
