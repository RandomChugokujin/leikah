---
title: Sudo
description: Exploit misconfigured sudo privileges to escalate privileges
categories: [Privilege Escalation]
tags: [Privilege Escalation, Linux, Sudo]
weight: 2
---
Sudo privileges can be granted to an account, permitting the account to run certain commands in the context of root or another account. When `sudo` is prepended to a command, the system will check if the user issuing the command has the appropriate rights as configured by the sudo policy, commonly `/etc/sudoers` and files included from `/etc/sudoers.d`.

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

From here, the goal is to **execute commands from the program we are allowed to run**, as the permitted run-as user (`root` in this example). We can use resources such as [GTFOBins](https://gtfobins.org/) to find shell escapes or other command-execution features, or research vulnerabilities in the installed version. Allowed arguments and security controls may restrict these techniques.

## Sudo Group

Similarly, membership in the `sudo` or `wheel` group may permit running any command as any user, but only if the sudo policy grants it. Check `sudo -l`.
```shell-session
johnadm@NIX02:~$ sudo -l

User johnadm may run the following commands on NIX02:
    (ALL : ALL) ALL
```

In that case, simply run `sudo su` to become root.

## Mitigation
1. Use absolute paths and restrict arguments in sudoers rules. Keep allowed binaries and their parent directories non-writable by unprivileged users; use `secure_path` for commands that invoke other programs through `PATH`. Absolute paths alone do not prevent shell escapes.
2. Grant sudo rights sparingly and based on the principle of least privilege.
3. Use correctly configured AppArmor profiles as defense in depth.
