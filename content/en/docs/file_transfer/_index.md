---
title: File Transfer
description: Learn how to transfer files from and to a compromised target.
categories: [File Transfer]
tags: [File Transfer]
weight: 2
---

After we compromise a host and gain command execution capabilities, we may want to transfer files such as enumeration scripts or exploits to the machine for privilege escalation, or we may wish to exfiltrate files from the machine that can further assist our engagement.

There are various services we can utilized to transfer files from and to a compromised target, some may seem more legitimate to the defenders than others. Depending on engagement type, stealth may be a consideration.

Another consideration may be whether the file is encrypted in transit. Encrypting files may be desirable if we want to avoid alarming the defenders, or the file may contain sensitive data that we can't avoid transferring.

Therefore, it is important to know as many methods of file transfer as possible so that we can pick one that best suit our needs across different engagements.
