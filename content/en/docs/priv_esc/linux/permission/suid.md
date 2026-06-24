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
