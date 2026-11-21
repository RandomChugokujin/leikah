---
title: SSH
description: Secure Shell
categories: [Services]
tags: [Services, SSH, Remote Management]
weight: 2
---

## Service Info
- Name: Secure Shell (SSH)
- Purpose: Encrypted network protocol
- Listening port: **22 (TCP)**
- OS: Unix-Like (more commonly), Windows

SSH is an encrypted network protocol used for secure remote management, file transfer, and tunneling. It replaces insecure protocols such as Telnet and the Berkeley r-commands.

The most commonly used SSH software is OpenSSH, which is developed by the OpenBSD developers. OpenSSH supports many authentication methods, including password and public-key authentication.

## Attack Flow
1. Identify SSH Version
2. Check whether password login or public-key authentication is enabled
3. Find SSH keys through other attack surfaces and use them to log in
    - If a key is protected by a passphrase, try cracking it with John the Ripper
4. Leverage file read vulnerabilities to read existing SSH keys or leverage file write vulnerabilities to write your own.
5. Login brute-forcing
    - Password spray/Credential stuff other valid credentials on the network if password login is enabled.
    - Try working SSH keys on other hosts if public-key authentication is enabled.

## Footprinting
Nmap Scan
```bash
sudo nmap -A -p22 <host>
```
OpenSSH supports password authentication, but passwords travel inside the encrypted SSH transport, not in plaintext on the network. Defaults vary by distribution and server policy; prefer public-key authentication and restrict direct root login.

We can see what login options are available by using the `-v` flag during login.
```shell-session
$ ssh -v brian@10.0.0.1
OpenSSH_8.2p1 Ubuntu-4ubuntu0.3, OpenSSL 1.1.1f  31 Mar 2020
debug1: Reading configuration data /etc/ssh/ssh_config
[...]
debug1: Authentications that can continue: publickey,password,keyboard-interactive
```
To request only password authentication from the client, use `-o PreferredAuthentications=password`. This cannot enable a method disabled by the server.
```bash
ssh -v <user>@<host> -o PreferredAuthentications=password
```

### Brute Forcing
We can use `hydra` to brute-force login.
- Use `-l` to specify username or `-L` to specify a username list.
- Use `-p` to specify password or `-P` to specify a password list.
- Use `-M` to specify a list of targets

```bash
hydra -L user.txt -p "password" ssh://10.0.0.1
```

Alternatively, we can also use `hydra -C` to credential stuff the SSH service with valid credentials we found elsewhere. We need to provide the filename of a list of colon-separated credentials (`username:password`).
```bash
hydra -C creds.txt ssh://10.0.0.1
```

## SSH Key
OpenSSH supports public-key authentication. Generate a key pair and share **ONLY the public key** with the server. The client signs session-bound authentication data with its private key; the server verifies the signature using an authorized public key. The private key is not sent to the server.

Common OpenSSH key types include RSA, Ed25519, and ECDSA, plus FIDO-backed variants. DSA is legacy: disabled by default since OpenSSH 7.0 and removed in OpenSSH 10.0.

To generate our own SSH keys, we can use the `ssh-keygen` utility, which prompts us for the key file path and an optional passphrase. The utility will generate a private key with the original provided filename, and a public key with a `.pub` extension.
```shell-session
$ ssh-keygen
Generating public/private ed25519 key pair.
Enter file in which to save the key (/home/brian/.ssh/id_ed25519): ./key
Enter passphrase for "./key" (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in ./key
Your public key has been saved in ./key.pub
The key fingerprint is:
SHA256:AjsqBEqppqzzy3DrVFckr0bayBOQTgGarFeOtHvuZro brian@rx-93-nu
The key's randomart image is:
+--[ED25519 256]--+
|..oo  . .        |
|o.+.   +         |
|+*. + . o        |
|*..* O o         |
|+o+ X * S        |
|=. + = .         |
|+.= .            |
|o* +o            |
|.+EBo            |
+----[SHA256]-----+
$ ls -l
total 8
-rw------- 1 brian wheel 411 Dec  4 12:40 key
-rw-r--r-- 1 brian wheel  96 Dec  4 12:40 key.pub
```
OpenSSH 9.5 and later default to Ed25519 keys (older versions defaulted to RSA). Use `-t` to choose a key algorithm; common options, subject to version/build support, include:
- `ecdsa`
- `ecdsa-sk`
- `ed25519`
- `ed25519-sk`
- `rsa`
```bash
ssh-keygen -t <algorithm>
```

Use the `-i` option for `ssh` to specify the path of the private key file.
```bash
ssh -i <key> <user>@<host>
```

### Adding Generated SSH Public Key to Server
If public-key login is enabled and the account is permitted to log in, appending our public key to its configured `AuthorizedKeysFile` (normally `$HOME/.ssh/authorized_keys`) may grant SSH access. Ownership, permissions, and key restrictions must also pass server checks.

This can either give us initial access from an arbitrary file write or establish persistence on an already compromised system.

```bash
echo "<public_key>" >> $HOME/.ssh/authorized_keys
```
Then we can use the associated private key to log in.

### Reading SSH Private Keys
If we find a file-read vulnerability, we can use it to read the user's SSH private keys. SSH private keys are commonly stored in `$HOME/.ssh/`, but may be elsewhere. Common default filenames reflect their key algorithm:
- `id_rsa`
- `id_ed25519`
- `id_ecdsa`
- `id_dsa`

After saving a private key, restrict access to its owner. `chmod 600` is a typical choice; mode `400` also works. OpenSSH rejects overly accessible private keys, not every mode other than `600`.
```bash
chmod 600 <ssh_key>
```

### SSH Key Passphrase Brute Forcing
SSH private keys may be protected via a passphrase. We can use John the Ripper, a CPU-based password cracker, to recover the passphrase.

We can first obtain the password hash by using the `ssh2john` script included in the John the Ripper Jumbo version, then use `john` to crack it.
```bash
ssh2john my_ssh_key > ssh_hash.txt
john --wordlist=wordlist.txt ssh_hash.txt
```

## File Transfer
See the article on [SSH File Transfer]({{% relref "/docs/file_transfer/ssh_file_transfer" %}}) for more details.

## References
- [OpenSSH ssh-keygen manual](https://man.openbsd.org/ssh-keygen) (key algorithms and default filenames).
- [Use John the Ripper to Crack SSH Private Keys](https://labex.io/tutorials/kali-use-john-the-ripper-to-crack-ssh-private-keys-594223)
