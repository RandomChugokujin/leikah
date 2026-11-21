---
title: SQL Injection
description: The classic web application attack to dump SQL databases and more.
categories: [Web Attack, SQL Injection]
tags: [Web Attack]
weight: 2
---

Web applications often interact with SQL databases to **Create, Read, Update, and Delete (CRUD)** data through SQL queries. **SQL injection** occurs when untrusted input changes the SQL query sent by the web application to the database. The input is interpreted as SQL syntax rather than data. A quote may terminate a string literal, but numeric injection may need no quote, and quote behavior depends on the DBMS and SQL mode.

Once injection has been established, the attacker can alter query logic, combine compatible `SELECT` results with `UNION`, or execute stacked statements if the DBMS and application driver permit them. Stacked statements are not required for SQL injection.

SQLi can have a significant impact, especially if database and server privileges are excessive. Sensitive data such as user credentials and payment information may be retrieved. SQL injection can also subvert application logic, bypassing authentication or accessing features restricted to specific users.

Prevent SQL injection with parameterized queries for data values. Use allow-listed choices for identifiers or keywords that cannot be bound as parameters; input validation alone is not a substitute. Apply least privilege to database accounts.
