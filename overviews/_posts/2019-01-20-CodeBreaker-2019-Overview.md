---
layout: post
permalink: /CodeBreaker-2019-Overview/
title: NSA Codebreaker 2019, Overview
---

#NSA Codebreaker 2019, Overview#
Each year NSA puts out a challenge called Codebreaker that requires reverse engineering and exploitation skills. This year was designed around Android Apps and Public Key backdoors. The theme for 2017 was incident response, 2018 was ransomware recovery, and this year was more intelligence operations based. 

The FICTITIOUS story is that a Terrorist's secure mobile chat application is covered and requires an analyst to examine it to prevent a future terrorist attack. The chat application encrypts messages using 2048 bit RSA keys. The analyst is provided login credintials and RSA key pair for one terrorist. The analyst must determine methods to map out the terrorist network, spoof messages from organization leader and finally decrypt all past messages. 

This year I was able to complete all eight tasks, along with 3.2% of participants who finished task 1. There was a 132% increase of people who signed up compared to last year but almost the same number of people who finished the first task. 

- 3779 people signed up 2019
- 1535 finished task 1. 2018 
- 50 finished task 7

- 2866 people signed up 2018
- 1536 finished task 0. 2018 


#Task 1 - It Begins! - [Getting Started - Part 1] - (Network Traffic Analysis)#
This was a simple task to extract an andorid app downloaded over HTTP. 

[Task 1 Writeup](https://armerj.github.io/CodeBreaker-2019-Task-1/)

#Task 2 - Permissions - [Getting Started - Part 2] - (Mobile APK Analysis)#
This task required analyzing the the Android app for permissions requested and the SHA256 of the signing certificate. 

Task 2 Writeup
[Task 2 Writeup](https://armerj.github.io/CodeBreaker-2019-Task-2/)

#Task 3 - Turn of Events - [Getting Started - Part 3] - (Database Analysis)#
This task required analyzing a SQLite, a common mobile database filetype, to determine what domains the application reaches out to. 

[Task 3 Writeup](https://armerj.github.io/CodeBreaker-2019-Task-3/)

#Task 4 - Schemes - (Cryptography; Reverse Engineering; Language Analysis)#
This task required analyzing TerrorTime to determine how to recover the login pin. Once the pin was recovered, the partipanct needed to login to analyze the messages sent to and from the captured terrorist. 

[Task 4 Writeup](https://armerj.github.io/CodeBreaker-2019-Task-4/)

#Task 5 - Masquerade - (Vulnerability Analysis)#
This task required the analyst to analyze the login method and determine a method to masquerade as any user using the application. Additionally, the partipanct needed to determine who was the organization's leader and recover his last encrypted message. 

[Task 5 Writeup](https://armerj.github.io/CodeBreaker-2019-Task-5/)

#Task 6a - Message Spoofing - (Vulnerability Analysis; Cryptanalysis)#
This task required the analyst to analyze how TerrorTime sent and recieved messages inorder to spoof messages from the orginzation's leader to a cell leader. The orginzation's leader should not be able to read any replies to the spoof messages. 

[Task 6a Writeup](https://armerj.github.io/CodeBreaker-2019-Task-6a/)

#Task 6b - Future Message Decryption - (Vulnerability Analysis; Cryptanalysis)#
This task required the analyst to determine a method to allow decryption of all future messages in TerrorTIme. 

[Task 6b Writeup](https://armerj.github.io/CodeBreaker-2019-Task-6b/)

#Task 7 - Distrust - (Reverse Engineering; Cryptography, Exploit Development)#
This was the final task and the most interesting to me. To ensure that the terrorist leadership could read any messages they put a backdoor in the RSA key generator code. The analyst was required to analyze the recovered keygen program, written in rust, to determine how the backdoor worked. Then, the analyst needed to recover all RSA private keys inorder to decrypt messages and determine who the terrorist's are targeting and any future plans. 

Coming Soon!
