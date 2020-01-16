---
layout: default
permalink: /CodeBreaker-2019-Task-1/
title: NSA Codebreaker 2019, Task 1
---

Task 1 requires you to extract the android app, TerrorTime, and any registration information from the provided PCAP. To complete this task you need to submit<br>
- the APK's SHA256 and<br>
- registration information for two users. <br>

I used Wireshark to extract the APK and a text file containing registration information from the PCAP. 

![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/task_1/export_menu.png)

![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/task_1/export.png)

Next, I used sha256sum to compute the APK's SHA256 hash. README.developer contained the required usernames. 
