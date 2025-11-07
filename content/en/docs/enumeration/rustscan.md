---
title: Rustscan
description: Scan ports faster with Rustscan
categories: [Enumeration, Service Enumeration]
tags: [Enumeration, Rustscan, Port-Scanning]
weight: 2
---

## [Rustscan](https://github.com/bee-san/RustScan)

Rustscan's project repo describes itself as a modern port scanner. It scans a large batch of ports asynchronously, reducing the overhead from threads and system calls. Thus achieving a scanning speed leagues ahead of Nmap. However, Rustscan is not a direct replacement for Nmap, as the former lacks much of the Service scanning capabilities. Rustscan would in fact feed the ports it found open during its scan into an Nmap scan, allowing the user to use Nmap for service enumeration or Nmap script scans.

## Basic Usage
For a basic run, use `-a` to specify the host(s), which accepts multiple types of arguments:

Single or comma-delimited list of IP addresses:
```bash
rustscan -a 127.0.0.1,0.0.0.0
```
Single or comma-delimited list of hostnames, or hostnames mixed with IP addresses
```bash
rustscan -a www.google.com, 127.0.0.1
```
CIDR subnets:
```bash
rustscan -a 192.168.0.0/30
```
Lastly, the filename of a list of hosts:

```bash
# hosts.txt:
192.168.0.1
192.168.0.2
google.com
192.168.0.0/30
127.0.0.1
```
```bash
rustscan -a 'hosts.txt'
```

## Specifying Ports
Use `-p` to specify individual ports or comma-delimited list of ports:
```bash
rustscan -a 127.0.0.1 -p 53
```
```bash
rustscan -a 127.0.0.1 -p 53,80,121,65535
```

Use `-r` to specify a range of ports
```bash
rustscan -a 127.0.0.1 --range 1-1000
```

## Nmap Arguments
Use the `--` to specify the arguments passed to the Nmap run Rustscan initiates after it finishes its own scan.

For example, the following Rustscan command:
```bash
rustscan -a 127.0.0.1 -- -A -sC
```
Runs the Nmap commnad:
```bash
nmap -Pn -vvv -p $PORTS -A -sC 127.0.0.1
```

## Performance Tuning
Since Rustscan is very aggressive out of the box, it could potentially trigger defenses to block your IP address. To prevent that from occurring, we can:
1. Decrease batch size: Use the `-b <number>` argument to specify a smaller batch size.
2. Increase timeout: Use the `-T <timeout>` argument to specify a longer timeout, in milliseconds, so that Rustscan would wait longer for each port.

## Config File
Rustscan accepts a TOML configuration file in the user's home directory, allowing the user to specify certain default arguments for each scan. The config file is read from `~/.rustscan.toml`.

The following options can be specified:
- `addresses`
- `ports`
- `range`
- `scan_order`
- `command`
- `accessible`
- `greppable`
- `batch-size`
- `timeout`
- `ulimit`

Example config:
```toml
addresses = ["127.0.0.1", "192.168.0.0/30", "www.google.com"]
command = ["-A"]
ports = {80 = 1, 443 = 1, 8080 = 1}
range = { start = 1, end = 10 }
greppable = false
accessible = true
scan_order = "Serial"
batch_size = 1000
timeout = 1000
tries = 3
ulimit = 1000
```

## References
- [Rustscan Project Github Repo and Wiki](https://github.com/bee-san/RustScan)
