---
title: SSH
description: Secure Shell
categories: [Services]
tags: [Services, SSH]
weight: 2
---

## Service Info
- Name: Secure Shell (SSH)
- Purpose: Encrypted network protocol
- Listening port: **22 (TCP)**
- OS: Unix-Like (more commonly), Windows

SSH is an ecrypted network protocol that is often used for secure network management, file transfer and tunneling that replaced unsecure protocol such as Telnet, Berkeley R-Suite protocols.

The most commonly used SSH software is OpenSSH, which is developed by the OpenBSD developers. OpenSSH supports many authentication methods, including password and public-key authentication.

## Attack Flow
1. Identify SSH Version
2. Check for if password login or public-key authentication is enabled
3. Find SSH keys in other attack surfaces and use them to login
    - If key protected by passphrase, try cracking with John the Ripper
4. Leverage file read vulnerabilities to read existing SSH keys or leverage file write vulnerabilities to write your own.
5. Login brute-forcing
    - Password spray/Credential stuff other valid credentials on the network if password login is enabled.
    - Try working SSH keys on other hosts if public-key authentication enabled.

## Footprinting
Nmap Scan
```bash
sudo nmap -A -p22 <host>
```
By default, OpenSSH allows plaintext password authentication, however, it's considered best practice to use only public key authentication and disable login for root user.

We can see what login options are available by using the `-v` flag during login.
```shell-session
$ ssh -v brian@10.0.0.1
OpenSSH_8.2p1 Ubuntu-4ubuntu0.3, OpenSSL 1.1.1f  31 Mar 2020
debug1: Reading configuration data /etc/ssh/ssh_config
[...]
debug1: Authentications that can continue: publickey,password,keyboard-interactive
```
To force the server to use password authentication, we use the `-o` flag to specify option `PreferredAUthentications`.
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

Alternatively, we can also use `hydra -C` to credential stuff the SSH service with valid credentials we found elsewhere. We need to provide the filename of a list of colon separated credentials (`username:password`).
```bash
hydra -C creds.txt ssh://10.0.0.1
```

## SSH Key
OpenSSH can be configured to use public-private key to login. To use public key login, the client must generate its own public-private key pair and share **ONLY the public key** to the server. During authentication, the server generates a cryptographic problem using the client's public key and sends it to the client. If the client can successfully decrypt the problem and send back the solution, the client is authenticated and granted access.

Currently, OpenSSH supports 4 common types of SSH keys:
- RSA
- Ed25519
- ECDSA
- DSA

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
By default, `ssh-keygen` generate keys using the `ed25519` protocol. We can use the `-t` argument to specify the algorithm of public key we want to generate. The available options are:
- `ecdsa`
- `ecdsa-sk`
- `ed25519`
- `ed25519-sk`
- `rsa`
```bash
ssh-keygen -t <algorithm>
```

Use `-i` option for `ssh` to specify the path of the private key file.
```bash
ssh -i <key> <user>@<host>
```

### Adding Generated SSH Public Key to Server
If we have file write ability to the server, we get access to SSH login by appending our public key to a user's `$HOME/.ssh/authorized_keys` file.

This can either give us initial access from a arbitrary file write or establish persistence on an already compromised system.

```bash
echo "<public_key>" >> $HOME/.ssh/authorized_keys
```
Then we can use the associated private key to login.

### Reading SSH Private Keys
If we found a file read vulnerability, we can use it to read the user's SSH private keys. Users' SSH private keys are stored on in `$HOME/.ssh/` directory, and can have one of the following default filenames, each corresponding to the public key encryption protocol they use:
- `id_rsa`
- `id_ed25519`
- `id_ecdsa`
- `id_dsa`

After reading the key and saving it to a file, the SSH client requires the permission on the file to be `600` (owner read-write only) before using it to login.
```bash
chmod 600 <ssh_key>
```

### SSH Key Passphrase Brute Forcing
SSH private keys may be protected via a passphrase. We can use John the Ripper, a CPU-based password cracker to recover the passphrase.

We can first obtain the password hash by using the `ssh2john` script included in the John the Ripper Jumbo version, then use `john` to crack it.
```bash
ssh2john my_ssh_key > ssh_hash.txt
john --wordlist=wordlist.txt ssh_hash.txt
```

## File Transfer
See the article on [SSH File Transfer]({{% relref "/docs/file_transfer/ssh_file_transfer" %}}) for more details.

## References
- Filenames of private SSH keys are obtained through ChatGPT.
- [Use John the Ripper to Crack SSH Private Keys](https://labex.io/tutorials/kali-use-john-the-ripper-to-crack-ssh-private-keys-594223)
