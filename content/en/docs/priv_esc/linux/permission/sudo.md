---
title: Sudo
description: Exploit misconfigured sudo privileges to escalation privileges
categories: [Privilege Escalation]
tags: [Privilege Escalation, Linux, Sudo]
weight: 2
---
Sudo privileges can be granted to an account, permitting the account to run certain commands in the context of root or another account. When `sudo` is prepended to a command, the system will check if the user issuing the command has the appropriate rights as configured in `/etc/sudoers` file.

Sudo privileges can be enumerated using `sudo -l`:
- Sometimes running this command requires us to provide the user's password.
- If an entry is marked with `NOPASSWD`, we can run the command without providing the user's password.
```shell-session
john@NIX02:~$ sudo -l

Matching Defaults entries for john on NIX02:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User john may run the following commands on NIX02:
    (root) NOPASSWD: /usr/sbin/tcpdump
```

From here, the goal is to **execute command from the program we are allowed to run**. We can make use of resources such as [GTFOBins](https://gtfobins.org/) to find options and other ways to execute command as `root`, or research vulnerabilities the specific version of the installed executable listed above.

## Sudo Group

Similarly, if a user belongs in `sudo` or `wheel` group, we may be allowed to run any command as any user.
```shell-session
johnadm@NIX02:~$ sudo -l

User johnadm may run the following commands on NIX02:
    (ALL : ALL) ALL
```

In that cause, simply run `sudo su` to become root.

## Mitigation
1. Always specify the absolute path to any binaries listed in the sudoers file entry. Otherwise, an attacker may be able to leverage PATH abuse.
2. Grant sudo rights sparingly and based on the principle of least privilege.
3. Use solutions such as AppArmor
