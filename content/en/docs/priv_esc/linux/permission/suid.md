---
title: setuid
description: Leverage executables with setuid permissions to escalate privileges
categories: [Privilege Escalation]
tags: [Privilege Escalation, Linux, setuid]
weight: 2
---
The **Set User ID upon Execution (setuid)** permission runs a binary with the effective user ID of its owner, typically with elevated privileges. Linux ignores setuid bits on scripts. The bit is also ignored on `nosuid` mounts or when `no_new_privs` is set.

We may use the following command to find `setuid` files owned by root. In `ls -l`, the owner execute position shows `s` (or `S` if owner execute permission is absent).
```bash
find / -type f -user root -perm -4000 -exec ls -ldb {} \; 2>/dev/null
```

If one of the executables listed above allows commands to be executed without dropping privileges, it may be leveraged to execute commands as root.
- We may use a resource like [GTFOBins](https://gtfobins.org/), or research vulnerabilities associated with the executable.

## Bash with Setuid
When run with `-p`, Bash retains its effective user ID instead of resetting it to the real user ID. Creating a copy of Bash owned by root with the setuid bit set can have two applications:
1. For creating a backdoor that allows any user on the system to run commands as root.
2. Turning the ability to create a root-owned executable and set its setuid bit on the host into the ability to execute commands as root. File-write access alone may not allow this.

With root-equivalent permissions, use the following commands to create a root-owned copy of Bash in the current directory, executable by everyone, with the setuid bit set. The host filesystem must allow execution and honor setuid; `-p` alone does not grant privileges.
```bash
cp "$(command -v bash)" ./bash
chown root:root ./bash
chmod 4755 ./bash
```

Then, simply run `./bash -p`:
```shell-session
$ ./bash -p
# whoami
root
```
