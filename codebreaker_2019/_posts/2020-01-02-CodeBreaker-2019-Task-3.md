---
layout: default
permalink: /CodeBreaker-2019-Task-3/
title: NSA Codebreaker 2019, Task 3
---

Task 3 requires you to examine the database TerrorTime uses. To complete this task you need to submit<br>
- IP address of the OAUTH server and <br>
- IP address of the XMPP server. <br>

I was provided a SQLite database that was recovered from a device with TerrorTime installed. I used [DB Browser for SQLite](https://sqlitebrowser.org/) to open an examine the database. Below is a brief explanation of the different columns. <br>
- Col 1 \- ClientID for OAuth<br>
- Col 2 \- Server IP for registration<br>
- Col 3 \- User's XMPP name<br>
- Col 4 \- XMPP Server IP<br>
- Col 5 \- Client secret (used for OAuth and encrypting)<br>
- Col 6 \- Access Token (Used for accessing XMPP Server)<br>
- Col 7 \- OAuth Renew Token<br>
- Col 8 \- Access Token expiration<br>
- Col 9 \- Renew Token expiration<br>
- Col 10 \- RSA Public Key<br>
- Col 11 \- Encrypted RSA Private Key<br>
- Col 12 \- Encrypted pin for accessing TerrorTime application<br>

![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/Task_3/database_code.png)

I found this information the ClientDBHandlerClass.addOrUpdateClient function. One thing to note for later is that the username appears twice; in column one and column 3. 

* image of database *
![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/Task_3/database.png)

The command line dig can resolve the domain name to an IP address. 
- chat[.]terrortime[.]app resolves to 54[.]91[.]5[.]130
- register[.]terrortime[.]app points to codebreaker[.]ltsnet[.]net which resolves to 54[.]197[.]185[.]236

![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/Task_3/dig_command.png)



