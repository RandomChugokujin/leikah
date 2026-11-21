---
title: Pivoting with Ligolo-NG
description: Use Ligolo-NG to tunnel into internal networks
categories: [Tunneling]
tags: [Tunneling, Ligolo]
weight: 2
---

[Ligolo-NG](https://github.com/nicocha30/ligolo-ng) is an offensive security tunneling tool. It uses a TUN network interface to route supported traffic to an internal network, providing VPN-like access without a SOCKS proxy.

Ligolo-NG is useful in labs that involve internal network pivoting. Check the current exam rules before using it in OSCP or CPTS.

## Creating a Tunnel
Suppose we have a pivot machine that shares a network with the attacker machine (**192.168.1.0/24**) and is connected to an internal network (**10.10.0.0/24**) that the attacker machine cannot access directly. Transfer the Ligolo agent to the pivot machine to forward traffic into the internal network.

![](/img/ligolo/network.png)

First, we create a TUN device on the attacker machine. This requires root or `CAP_NET_ADMIN`; the agent does not. Newer versions (>= 0.6) also support `interface_create` in the proxy CLI.
```bash
sudo ip tuntap add user $USER mode tun ligolo
sudo ip link set ligolo up
```

Now, we launch the Ligolo-NG proxy on our attacker machine, which listens on port 11601 by default. The capture uses a binary named `ligolo-ng`; upstream release archives call it `proxy`. CLI names vary by release: current docs use `tunnel_start --tun ligolo`, while the captures below use `start`. Check `help` for your version.
```shell-session
╭─brian@rx-93-nu ~
╰─$ ligolo-ng -selfcert
INFO[0000] Loading configuration file ligolo-ng.yaml
WARN[0000] Using default selfcert domain 'ligolo', beware of CTI, SOC and IoC!
INFO[0000] Listening on 0.0.0.0:11601
    __    _             __
   / /   (_)___ _____  / /___        ____  ____ _
  / /   / / __ `/ __ \/ / __ \______/ __ \/ __ `/
 / /___/ / /_/ / /_/ / / /_/ /_____/ / / / /_/ /
/_____/_/\__, /\____/_/\____/     /_/ /_/\__, /
        /____/                          /____/

  Made in France ♥            by @Nicocha30!
  Version: dev

ligolo-ng »
```

Next, we transfer the Ligolo Agent to the target. Versions for Linux and Windows are available, and their command syntax is the same. Root or SYSTEM privilege is not required to run the agent.
```cmd
.\agent.exe -connect <attacker_ip>:11601 -ignore-cert
```
```bash
./agent -connect <attacker_ip>:11601 -ignore-cert
```

The `-ignore-cert` examples disable TLS certificate verification and are for isolated labs only. Prefer `-accept-fingerprint <fingerprint>`, using the value from the proxy's `certificate_fingerprint` command verified through a trusted channel.

On our Ligolo CLI interface, we see that the pivot machine has connected. We may use the `session` command to select an agent for routing traffic.
```shell-session
ligolo-ng » INFO[0388] Agent joined.                                 id=bc24115dfae8 name=brian@pivot01 remote="192.168.1.29:42724"
ligolo-ng » session
? Specify a session : 1 - brian@pivot01 - 192.168.1.29:42724 - bc24115dfae8
[Agent : brian@pivot01] »
```
Use the `ifconfig` command to list the interfaces available on the pivot host.
```shell-session
[Agent : brian@pivot01] » ifconfig
┌────────────────────────────────────┐
│ Interface 0                        │
├──────────────┬─────────────────────┤
│ Name         │ lo                  │
│ Hardware MAC │                     │
│ MTU          │ 65536               │
│ Flags        │ up|loopback|running │
│ IPv4 Address │ 127.0.0.1/8         │
│ IPv6 Address │ ::1/128             │
└──────────────┴─────────────────────┘
┌───────────────────────────────────────────────┐
│ Interface 1                                   │
├──────────────┬────────────────────────────────┤
│ Name         │ ens18                          │
│ Hardware MAC │ bc:24:11:5d:fa:e8              │
│ MTU          │ 1500                           │
│ Flags        │ up|broadcast|multicast|running │
│ IPv4 Address │ 192.168.1.29/24                │
│ IPv6 Address │ fe80::796:c5b4:e859:2f70/64    │
└──────────────┴────────────────────────────────┘
┌───────────────────────────────────────────────┐
│ Interface 2                                   │
├──────────────┬────────────────────────────────┤
│ Name         │ ens19                          │
│ Hardware MAC │ bc:24:11:a7:95:60              │
│ MTU          │ 1500                           │
│ Flags        │ up|broadcast|multicast|running │
│ IPv4 Address │ 10.10.0.100/24                 │
│ IPv6 Address │ fe80::75b4:2baf:8f67:9469/64   │
└──────────────┴────────────────────────────────┘
┌───────────────────────────────────────┐
│ Interface 3                           │
├──────────────┬────────────────────────┤
│ Name         │ docker0                │
│ Hardware MAC │ 02:42:e0:07:87:5d      │
│ MTU          │ 1500                   │
│ Flags        │ up|broadcast|multicast │
│ IPv4 Address │ 172.17.0.1/16          │
└──────────────┴────────────────────────┘
┌───────────────────────────────────────┐
│ Interface 4                           │
├──────────────┬────────────────────────┤
│ Name         │ br-63b3c727249f        │
│ Hardware MAC │ 02:42:5d:7c:b8:4e      │
│ MTU          │ 1500                   │
│ Flags        │ up|broadcast|multicast │
│ IPv4 Address │ 172.19.0.1/16          │
└──────────────┴────────────────────────┘
```
In a separate terminal, create a route for the internal network (`10.10.0.0/24`) through the TUN interface.
```bash
sudo ip route add 10.10.0.0/24 dev ligolo
```
Use `ip route list` to check the route. The captured `linkdown` flag can appear before the tunnel starts; the route alone does not confirm connectivity.
```shell-session
╭─brian@rx-93-nu ~
╰─$ ip route list | grep ligolo
10.10.0.0/24 dev ligolo scope link linkdown
```
To start tunneling, we simply send the `start` command inside the Ligolo-CLI.
```shell-session
[Agent : brian@pivot01] » start
INFO[0776] Starting tunnel to brian@pivot01 (bc24115dfae8)
```

We can run commands and programs, and supported TCP, UDP and ICMP echo traffic to the routed network will be forwarded through the pivot. Ligolo-NG translates packets to agent-side sockets; it is not a raw-packet VPN. For Nmap, use `--unprivileged` (for example, with `-sT -Pn -n`) to avoid unsupported raw-packet assumptions.
```shell-session
╭─brian@rx-93-nu ~
╰─$ ping 10.10.0.3 -c 3
PING 10.10.0.3 (10.10.0.3) 56(84) bytes of data.
64 bytes from 10.10.0.3: icmp_seq=1 ttl=64 time=6.36 ms
64 bytes from 10.10.0.3: icmp_seq=2 ttl=64 time=3.86 ms
64 bytes from 10.10.0.3: icmp_seq=3 ttl=64 time=3.79 ms

--- 10.10.0.3 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 3.785/4.668/6.357/1.194 ms
```

## Create Listeners
To let an internal host reach our attacker machine through the pivot host, use `listener_add` in the Ligolo CLI. This creates a listener on the pivot host, useful for receiving reverse shells or letting internal hosts download files hosted on the attacker machine.
```shell-session
listener_add --addr 0.0.0.0:<listen_port> --to 127.0.0.1:<forward_port> --tcp
```
In the example below, connections to the listener on TCP port 8000 of the pivot machine are relayed to `127.0.0.1:8000` on the attacker machine. The pivot firewall must allow the listening port.
```shell-session
[Agent : brian@pivot01] » listener_add --addr 0.0.0.0:8000 --to 127.0.0.1:8000 --tcp
INFO[1262] Listener 0 created on remote agent!
[Agent : brian@pivot01] » listener_list
┌────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Active listeners                                                                                                           │
├───┬───────────────────────────────────────────────────┬─────────┬────────────────────────┬────────────────────────┬────────┤
│ # │ AGENT                                             │ NETWORK │ AGENT LISTENER ADDRESS │ PROXY REDIRECT ADDRESS │ STATUS │
├───┼───────────────────────────────────────────────────┼─────────┼────────────────────────┼────────────────────────┼────────┤
│ 0 │ brian@pivot01 - 192.168.1.29:42724 - bc24115dfae8 │ tcp     │ 0.0.0.0:8000           │ 127.0.0.1:8000         │ Online │
└───┴───────────────────────────────────────────────────┴─────────┴────────────────────────┴────────────────────────┴────────┘
```
To access the attacker host from the internal network via the listener, we specify the IP address of the pivot host in lieu of the IP address of the attacker machine.
```shell-session
PS C:\research> iwr http://10.10.0.100:8000/test.txt -Outfile test.txt
PS C:\research> type test.txt
Ligolo listener test
```

## Multi-Layered Tunneling
Ligolo may also be used to tunnel multiple layers of internal networks. Suppose we have another internal network, and another pivot host that has access to both the first internal network (`10.10.0.0/24`) and the second internal network (`172.16.0.0/24`).

To use Ligolo for multi-layered tunneling, we first create a listener on port 11601 of **pivot01** that forwards traffic back to port 11601 of the attacker machine.
```shell-session
[Agent : brian@pivot01] » listener_add --addr 0.0.0.0:11601 --to 127.0.0.1:11601 --tcp
INFO[2263] Listener 1 created on remote agent!
[Agent : brian@pivot01] » listener_list
┌────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ Active listeners                                                                                                           │
├───┬───────────────────────────────────────────────────┬─────────┬────────────────────────┬────────────────────────┬────────┤
│ # │ AGENT                                             │ NETWORK │ AGENT LISTENER ADDRESS │ PROXY REDIRECT ADDRESS │ STATUS │
├───┼───────────────────────────────────────────────────┼─────────┼────────────────────────┼────────────────────────┼────────┤
│ 0 │ brian@pivot01 - 192.168.1.29:42724 - bc24115dfae8 │ tcp     │ 0.0.0.0:8000           │ 127.0.0.1:8000         │ Online │
│ 1 │ brian@pivot01 - 192.168.1.29:42724 - bc24115dfae8 │ tcp     │ 0.0.0.0:11601          │ 127.0.0.1:11601        │ Online │
└───┴───────────────────────────────────────────────────┴─────────┴────────────────────────┴────────────────────────┴────────┘
```
Now, we transfer and run the agent on the second pivot host (`pivot02`), connecting back to `pivot01`.
```cmd
.\ligolo-agent.exe -connect 10.10.0.100:11601 -ignore-cert
```
On the Ligolo-CLI, we switch to session 2 on the newly joined `pivot02`.
```shell-session
[Agent : brian@pivot01] » session
? Specify a session : 2 - PIVOT02\\brian@PIVOT02 - 127.0.0.1:43778 - 005056b057d5
[Agent : PIVOT02\\brian@PIVOT02] » ifconfig
┌───────────────────────────────────────────────┐
│ Interface 0                                   │
├──────────────┬────────────────────────────────┤
│ Name         │ Ethernet0                      │
│ Hardware MAC │ 00:50:56:b0:57:d5              │
│ MTU          │ 1500                           │
│ Flags        │ up|broadcast|multicast|running │
│ IPv6 Address │ fe80::51a8:46a:6c23:ad3e/64    │
│ IPv4 Address │ 10.10.0.3/24                   │
└──────────────┴────────────────────────────────┘
┌───────────────────────────────────────────────┐
│ Interface 1                                   │
├──────────────┬────────────────────────────────┤
│ Name         │ Ethernet1 2                    │
│ Hardware MAC │ 00:50:56:b0:72:e1              │
│ MTU          │ 1500                           │
│ Flags        │ up|broadcast|multicast|running │
│ IPv6 Address │ fe80::68d4:7c60:3edd:646c/64   │
│ IPv4 Address │ 172.16.0.35/24                 │
└──────────────┴────────────────────────────────┘
┌──────────────────────────────────────────────┐
│ Interface 2                                  │
├──────────────┬───────────────────────────────┤
│ Name         │ Loopback Pseudo-Interface 1   │
│ Hardware MAC │                               │
│ MTU          │ -1                            │
│ Flags        │ up|loopback|multicast|running │
│ IPv6 Address │ ::1/128                       │
│ IPv4 Address │ 127.0.0.1/8                   │
└──────────────┴───────────────────────────────┘
```
In a separate terminal, create a second TUN device named `ligolo-1` (the name is arbitrary) and add a route for `172.16.0.0/24` through it. Replace `brian` with the local user running the proxy.
```bash
sudo ip tuntap add user brian mode tun ligolo-1
sudo ip link set ligolo-1 up
sudo ip route add 172.16.0.0/24 dev ligolo-1
```

We may now start tunneling to the second internal network (`172.16.0.0/24`) with the `start` Ligolo-CLI command. We use the `--tun` option to specify the TUN device we created for the second layer of tunneling.
```shell-session
[Agent : PIVOT02\\brian@PIVOT02] » start --tun ligolo-1
INFO[4062] Starting tunnel to PIVOT02\\brian@PIVOT02 (005056b057d5)
```

Now we can forward our traffic across two hops and reach hosts on the second internal network.
