---
title: SOCKS Tunneling with Chisel
description: Use Chisel to tunnel traffic via HTTP.
categories: [Tunneling]
tags: [Tunneling, Chisel]
weight: 2
---

[Chisel](https://github.com/jpillora/chisel) is a TCP/UDP tunneling tool transported via HTTP and secured with SSH. It can help create a client-server tunnel in a firewall-restricted environment. Chisel can create a SOCKS proxy for applications configured to use it; it does not route all system traffic automatically.

## Standard Tunneling
After downloading or building the binary, transfer `chisel` to the pivot host and run it as a server.
```bash
./chisel server -v -p <server_port> --socks5
```

Now, run `chisel` on the attacker machine in client mode to connect to the pivot's HTTP listening port (separate from the local SOCKS port 1080).
```bash
./chisel client -v <pivot_ip>:<server_port> socks
```

## Reverse Tunneling
If firewall rules restrict inbound connections to the pivot machine, we can use Chisel to establish a reverse connection instead where we first start our server on the attacker machine.
```bash
./chisel server --reverse -v -p <server_port>
```

Then, connect as a client from the pivot machine.
```bash
./chisel client -v <attacker_ip>:<server_port> R:127.0.0.1:1080:socks
```

## Using the SOCKS Proxy
**Proxychains** may be used to route compatible applications' TCP connections through the SOCKS proxy established by Chisel. With the commands above, whether Chisel is run in standard or reverse mode, a SOCKS5 listener is established on `127.0.0.1:1080` of the attacker machine.

At the very end of `/etc/proxychains.conf` is a list of proxies Proxychains will attempt to use in sequence. We configure Proxychains to make use of the Chisel local SOCKS5 listener like so:
```txt
[ProxyList]
socks5 127.0.0.1 1080
```

Now, we may prepend `proxychains` to compatible dynamically linked applications using TCP connections. It does not proxy raw-packet scans, ICMP, or UDP. The config may be `/etc/proxychains4.conf` on newer installations; replace any unwanted existing proxy entries. Optionally, we use `-q` to have `proxychains` operate in quiet mode, suppressing output regarding tunnel connections.
```bash
proxychains -q xfreerdp /v:10.10.0.4 /u:amuro.ray /p:Password1
```


Use matching Chisel versions and verify the server fingerprint with the client's `--fingerprint` option through a trusted channel. Chisel supports fixed UDP forwards, but the `socks` remote used here is TCP-only. Newer versions document a separate `udpsocks` remote for SOCKS UDP support; this does not add UDP support to Proxychains.
