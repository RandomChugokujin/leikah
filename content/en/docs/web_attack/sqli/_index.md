---
title: SQL Injection
description: The classic web application attack to dump SQL databases and more.
categories: [Web Attack, SQL Injection]
tags: [Web Attack]
weight: 2
---

Web applications often interact with SQL databases to **Create, Read, Update, and Delete (CRUD)** data through SQL queries. **SQL injection** occurs when a malicious user attempts to pass input that changes the SQL query sent by the web application to the database. First, attacker has to inject code outside the expected user limit so it does not get interpreted as user input. This is accomplished by using a single or double quote to escape the limits of user input.

Once injection has been established, the attacker have to look for a way to execute a different SQL statement. This can be done using SQL code to make up a working query that executes both the intended and new SQL queries via either stacker queries or Union queries.

SQLi can have a tremendous impact, especially if privileges on the back-end server and database are very lax. Sensitive information and secrets like user logins and payment information may be retrieved. SQL injection can also be used to subvert intended web application logic such as bypassing login without valid credentials as well as accessing features locked to specific users.

Common ways to mitigate against SQL injection include validation and sanitization of user input before they are included in SQL queries and the use of parameterized queries.
