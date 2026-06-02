---
title: Linux Post-Exploit Enum
description: Gather information necessary for privilege escalation on Linux
categories: [Privilege Escalation]
tags: [Privilege Escalation, Linux Privilege Escalation]
weight: 2
---

Enumeration is key to privilege escalation. After gaining initial shell access, it's important to check for the following details:
- **OS Version**
- **Kernel Version**
- **Running Services**

## Situational Awareness
After getting a shell onto a Linux target, we should try to run a few commands to orient ourselves:
- `whoami`: username
- `id`: group membership
- `hostname`: server hostname (naming convention?)
- `ifconfig` / `ip a`: Network address(es)?
- `sudo -l`: Sudo permissions? With or without password?

## Operating System Information

### Linux Distribution
We also want to know what Linux Distribution is running on the target. Different distros have different tools and utilities available on the system that may aid our privilege escalation.
```shell-session
ubuntu@ubuntu:~$ cat /etc/*release*
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=25.04
DISTRIB_CODENAME=plucky
DISTRIB_DESCRIPTION="Ubuntu 25.04"
PRETTY_NAME="Ubuntu 25.04"
NAME="Ubuntu"
VERSION_ID="25.04"
VERSION="25.04 (Plucky Puffin)"
VERSION_CODENAME=plucky
ID=ubuntu
ID_LIKE=debian
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
UBUNTU_CODENAME=plucky
LOGO=ubuntu-logo
```
Alternatively, we can also check `/etc/*issue`
```shell-session
ubuntu@ubuntu:~$ cat /etc/issue
Ubuntu 25.04 \n \l
```

### Kernel Version
Kernel version can help us look up known vulnerabilities (DirtyPipe, CopyFail, etc.)
```shell-session
ubuntu@ubuntu:~$ uname -a
Linux ubuntu 6.14.0-15-generic #15-Ubuntu SMP PREEMPT_DYNAMIC Sun Apr  6 15:05:05 UTC 2025 x86_64 x86_64 x86_64 GNU/Linux
```

## Environment Information

### Environment Variables
Check the current environment variables:
```shell-session
ubuntu@ubuntu:~$ env
SHELL=/bin/bash
PWD=/home/ubuntu
LOGNAME=ubuntu
HOME=/home/ubuntu
LS_COLORS=rs=0:di=01;34:ln=01;36:mh=00:pi=40;33:so=01;35:do=01;35:bd=40;33;01:cd=40;33;01:or=40;31;01:mi=00:su=37;41:sg=30;43:ca=00:tw=30;42:ow=34;42:st=37;44:ex=01;32:*.7z=01;31:*.ace=01;31:*.alz=01;31:*.apk=01;31:*.arc=01;31:*.arj=01;31:*.bz=01;31:*.bz2=01;31:*.cab=01;31:*.cpio=01;31:*.crate=01;31:*.deb=01;31:*.drpm=01;31:*.dwm=01;31:*.dz=01;31:*.ear=01;31:*.egg=01;31:*.esd=01;31:*.gz=01;31:*.jar=01;31:*.lha=01;31:*.lrz=01;31:*.lz=01;31:*.lz4=01;31:*.lzh=01;31:*.lzma=01;31:*.lzo=01;31:*.pyz=01;31:*.rar=01;31:*.rpm=01;31:*.rz=01;31:*.sar=01;31:*.swm=01;31:*.t7z=01;31:*.tar=01;31:*.taz=01;31:*.tbz=01;31:*.tbz2=01;31:*.tgz=01;31:*.tlz=01;31:*.txz=01;31:*.tz=01;31:*.tzo=01;31:*.tzst=01;31:*.udeb=01;31:*.war=01;31:*.whl=01;31:*.wim=01;31:*.xz=01;31:*.z=01;31:*.zip=01;31:*.zoo=01;31:*.zst=01;31:*.avif=01;35:*.jpg=01;35:*.jpeg=01;35:*.mjpg=01;35:*.mjpeg=01;35:*.gif=01;35:*.bmp=01;35:*.pbm=01;35:*.pgm=01;35:*.ppm=01;35:*.tga=01;35:*.xbm=01;35:*.xpm=01;35:*.tif=01;35:*.tiff=01;35:*.png=01;35:*.svg=01;35:*.svgz=01;35:*.mng=01;35:*.pcx=01;35:*.mov=01;35:*.mpg=01;35:*.mpeg=01;35:*.m2v=01;35:*.mkv=01;35:*.webm=01;35:*.webp=01;35:*.ogm=01;35:*.mp4=01;35:*.m4v=01;35:*.mp4v=01;35:*.vob=01;35:*.qt=01;35:*.nuv=01;35:*.wmv=01;35:*.asf=01;35:*.rm=01;35:*.rmvb=01;35:*.flc=01;35:*.avi=01;35:*.fli=01;35:*.flv=01;35:*.gl=01;35:*.dl=01;35:*.xcf=01;35:*.xwd=01;35:*.yuv=01;35:*.cgm=01;35:*.emf=01;35:*.ogv=01;35:*.ogx=01;35:*.aac=00;36:*.au=00;36:*.flac=00;36:*.m4a=00;36:*.mid=00;36:*.midi=00;36:*.mka=00;36:*.mp3=00;36:*.mpc=00;36:*.ogg=00;36:*.ra=00;36:*.wav=00;36:*.oga=00;36:*.opus=00;36:*.spx=00;36:*.xspf=00;36:*~=00;90:*#=00;90:*.bak=00;90:*.crdownload=00;90:*.dpkg-dist=00;90:*.dpkg-new=00;90:*.dpkg-old=00;90:*.dpkg-tmp=00;90:*.old=00;90:*.orig=00;90:*.part=00;90:*.rej=00;90:*.rpmnew=00;90:*.rpmorig=00;90:*.rpmsave=00;90:*.swp=00;90:*.tmp=00;90:*.ucf-dist=00;90:*.ucf-new=00;90:*.ucf-old=00;90:
SSH_CONNECTION=192.168.122.1 57306 192.168.122.240 22
LESSCLOSE=/usr/bin/lesspipe %s %s
TERM=st-256color
LESSOPEN=| /usr/bin/lesspipe %s
USER=ubuntu
SHLVL=1
SSH_CLIENT=192.168.122.1 57306 22
DEBUGINFOD_URLS=https://debuginfod.ubuntu.com
XDG_DATA_DIRS=/usr/share/gnome:/usr/local/share:/usr/share:/var/lib/snapd/desktop
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/snap/bin
MAIL=/var/mail/ubuntu
SSH_TTY=/dev/pts/1
_=/usr/bin/env
```

Specifically, it may be beneficial to note the `PATH` environment variable. If any of the paths are writable by our user, we may be able to use PATH hijacking to achieve privilege escalation.
```shell-session
ubuntu@ubuntu:~$ env | grep PATH
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/snap/bin
```

### Login Shells
`/etc/shells` stores a list of valid login shells.
```shell-session
ubuntu@ubuntu:~$ cat /etc/shells
# /etc/shells: valid login shells
/bin/sh
/usr/bin/sh
/bin/bash
/usr/bin/bash
/bin/rbash
/usr/bin/rbash
/usr/bin/dash
```

## Users and Groups

### `/etc/passwd`
All users on the system are stored in `/etc/passwd`. Despite its name, the file no longer store password information on all modern Linux systems.

Structure of `/etc/passwd` entry:

| 1     | 2  | 3  | 4  | 5     | 6     | 7          |
|-------|----|----|----|-------|-------|------------|
| root: | x: | 0: | 0: | root: | /root:| /bin/bash |

1. Login name
2. Passwword (x: password in shadow file, *:user cannot login)
3. User ID (UID)
4. Primary Group ID (GID)
5. Comment Field/User full name
5. User home directory path
7. User default login shell

Interactive users can be found from `/etc/passwd` by grepping `sh$` for valid login shells.
```shell-session
ubuntu@ubuntu:~$ grep "sh$" /etc/passwd

root:x:0:0:root:/root:/bin/bash
jsmith:x:1000:1000:mrb3n:/home/jsmith:/bin/bash
bjones:x:1001:1001::/home/bjones:/bin/zsh
ubuntu:x:1002:1002::/home/ubuntu:/bin/bash
```

### Logged-on Users
Check currently on the system with commands such as `w`, `who` or `finger`.
```shell-session
ubuntu@ubuntu:~$ w

 12:27:21 up 1 day, 16:55,  1 user,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
jsmith   pts/0    10.10.14.16      Tue19   40:54m  0.02s  0.02s -bash
```

### Group Information
The `/etc/groups` file shows a list of groups on the system as well as their members:
```shell-session
ubuntu@ubuntu:~$ cat /etc/groups

root:x:0:
daemon:x:1:
bin:x:2:
sys:x:3:
adm:x:4:syslog,htb-student
tty:x:5:syslog
disk:x:6:
[...]
```

Individual group membership can also be queried using `getent`:
```shell-session
ubuntu@ubuntu:~$ getent group sudo

sudo:x:27:bjones
```

We should also check for users with a home directory under `/home`. This can help us find SSH keys, interesting information in shell history, and much more.
```shell-session
ubuntu@ubuntu:~$ ls /home

bjones      jsmith      ubuntu
```

## Network Information

### Network Interfaces
Start by checking available interfaces using `ip a`:
```shell-session
ubuntu@ubuntu:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute
       valid_lft forever preferred_lft forever
2: enp1s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 52:54:00:a8:01:02 brd ff:ff:ff:ff:ff:ff
    altname enx525400a80102
    inet 192.168.122.240/24 brd 192.168.122.255 scope global dynamic noprefixroute enp1s0
       valid_lft 3219sec preferred_lft 3219sec
    inet6 fe80::5054:ff:fea8:102/64 scope link proto kernel_ll
       valid_lft forever preferred_lft forever
```
Or `ifconfig`
```shell-session
ubuntu@ubuntu:~$ ifconfig
enp1s0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.122.240  netmask 255.255.255.0  broadcast 192.168.122.255
        inet6 fe80::5054:ff:fea8:102  prefixlen 64  scopeid 0x20<link>
        ether 52:54:00:a8:01:02  txqueuelen 1000  (Ethernet)
        RX packets 9861  bytes 39958679 (39.9 MB)
        RX errors 0  dropped 1111  overruns 0  frame 0
        TX packets 7670  bytes 820143 (820.1 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 183  bytes 17353 (17.3 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 183  bytes 17353 (17.3 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```
Note IP addresses in on different interfaces, as well as subnet mask.

### Network Routes
```shell-session
ubuntu@ubuntu:~$ route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         _gateway        0.0.0.0         UG    100    0        0 enp1s0
192.168.122.0   0.0.0.0         255.255.255.0   U     100    0        0 enp1s0
```

### ARP Information
Are there any other hosts the target has been connecting to?
```shell-session
ubuntu@ubuntu:~$ arp -a
_gateway (192.168.122.1) at 52:54:00:03:21:e5 [ether] on enp1s0
```

### Name Resolution
Check `/etc/hosts`, are there interesting hosts specified?
```shell-session
ubuntu@ubuntu:~$ cat /etc/hosts
127.0.0.1 localhost
127.0.1.1 ubuntu
10.10.0.3 ra-cailumn.gundam.local gundam.local

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
ff02::3 ip6-allhosts
```

Interesting nameservers under `/etc/resolv.conf`?
```shell-session
ubuntu@ubuntu:~$ cat /etc/resolv.conf
# This is /run/systemd/resolve/stub-resolv.conf managed by man:systemd-resolved(8).
# Do not edit.
#
# This file might be symlinked as /etc/resolv.conf. If you're looking at
# /etc/resolv.conf and seeing this text, you have followed the symlink.
#
# This is a dynamic resolv.conf file for connecting local clients to the
# internal DNS stub resolver of systemd-resolved. This file lists all
# configured search domains.
#
# Run "resolvectl status" to see details about the uplink DNS servers
# currently in use.
#
# Third party programs should typically not access this file directly, but only
# through the symlink at /etc/resolv.conf. To manage man:resolv.conf(5) in a
# different way, replace this symlink by a static file or a different symlink.
#
# See man:systemd-resolved.service(8) for details about the supported modes of
# operation for /etc/resolv.conf.

nameserver 127.0.0.53
options edns0 trust-ad
search .

```

Ubuntu, Fedora, and other popular Linux distros may use **Systemd-Resolved** to manage name resolution. If you see the contents like show above inside `/etc/resolv.conf`, we can further enumerate name resolution configuration:
```shell-session
ubuntu@ubuntu:~$ resolvectl status
Global
         Protocols: -LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
  resolv.conf mode: stub

Link 2 (enp1s0)
    Current Scopes: DNS
         Protocols: +DefaultRoute -LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
Current DNS Server: 192.168.122.1
       DNS Servers: 192.168.122.1
     Default Route: yes
```

## Packages and Utilities
List installed packages and note outdated packages.
```bash
apt list --installed | tr "/" " " | cut -d" " -f1,3 | sed 's/[0-9]://g' | tee -a installed_pkgs.list
```

Check for executables installed against [GTFObins](https://gtfobins.github.io/)
```bash
for i in $(curl -s https://gtfobins.org/api.json | jq -r '.executables | keys[]'); do if grep -q "$i" installed_pkgs.list; then echo "Check for GTFO: $i";fi; done
```

Sudo version information, useful for looking up known privilege escalation vulnerabilities.
```shell-session
ubuntu@ubuntu:~$ sudo -V

Sudo version 1.8.31
Sudoers policy plugin version 1.8.31
Sudoers file grammar version 46
Sudoers I/O plugin version 1.8.31
```

## Service Configuration

### SSHD
Check `/etc/ssh/sshd_config` for the SSH daemon configuration:
- Is root allowed to login?
- Cleartext passwords allowed?
- Any users disallowed from login via SSH?

### Web Servers
Check for virtual hosts that weren't discovered before.

Apache2 configs live under `/etc/apache2`:
```txt
/etc/apache2/
├── apache2.conf
├── ports.conf
├── sites-available/
├── sites-enabled/
├── conf-available/
├── conf-enabled/
├── mods-available/
└── mods-enabled/
```

Nginx configs live under `/etc/nginx`:
```txt
/etc/nginx/
├── nginx.conf
├── conf.d/
├── sites-available/
└── sites-enabled/
```

## Hardware Information

### CPU Information
Running `lscpu` can help us identify the CPU architecture. This can become more important as ARM-based CPUs become more prevalent.
```shell-session
ubuntu@ubuntu:~$ lscpu
Architecture:             x86_64
  CPU op-mode(s):         32-bit, 64-bit
  Address sizes:          48 bits physical, 48 bits virtual
  Byte Order:             Little Endian
CPU(s):                   2
  On-line CPU(s) list:    0,1
Vendor ID:                AuthenticAMD
  Model name:             AMD Ryzen 7 5800X3D 8-Core Processor
    CPU family:           25
    Model:                33
    Thread(s) per core:   1
    Core(s) per socket:   1
    Socket(s):            2
[...]
```

### Disk Information
Run `lsblk` to see block devices available on the system and their mount points if they are mounted.
- In Ubuntu systems, `lsblk` also shows the Snap packages installed as loop devices
```shell-session
ubuntu@ubuntu:~$ lsblk

NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
loop0                       7:0    0   55M  1 loop /snap/core18/1705
loop1                       7:1    0   69M  1 loop /snap/lxd/14804
loop2                       7:2    0   47M  1 loop /snap/snapd/16292
loop3                       7:3    0  103M  1 loop /snap/lxd/23339
loop4                       7:4    0   62M  1 loop /snap/core20/1587
loop5                       7:5    0 55.6M  1 loop /snap/core18/2538
sda                         8:0    0   20G  0 disk
├─sda1                      8:1    0    1M  0 part
├─sda2                      8:2    0    1G  0 part /boot
└─sda3                      8:3    0   19G  0 part
  └─ubuntu--vg-ubuntu--lv 253:0    0   18G  0 lvm  /
sr0                        11:0    1  908M  0 rom
```

If there are any network drives mounted at boot, we can check `/etc/fstab` to see if there are credentials listed.
```shell-session
ubuntu@ubuntu:~$ cat /etc/fstab

# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
# / was on /dev/ubuntu-vg/ubuntu-lv during curtin installation
/dev/disk/by-id/dm-uuid-LVM-BdLsBLE4CvzJUgtkugkof4S0dZG7gWR8HCNOlRdLWoXVOba2tYUMzHfFQAP9ajul / ext4 defaults 0 0
# /boot was on /dev/sda2 during curtin installation
/dev/disk/by-uuid/20b1770d-a233-4780-900e-7c99bc974346 /boot ext4 defaults 0 0
```
