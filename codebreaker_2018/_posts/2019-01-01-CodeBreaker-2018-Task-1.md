---
layout: default
permalink: /CodeBreaker-2018-Task-1/
title: NSA Codebreaker 2018, Task 1
---

For task 1, you are asked to find:<br>
- Victim Identifier<br>
- Encrypted Ransom Key<br>
- One-Time Passcode (OTP) used to authenticate the Ransom contract to the Register contract<br>
- Escrow contract address<br>

The Victim Identifier and the Escrow contract address can be found by examining the ransom note left by the ransomware. 

![_config.yml]({{ site.baseurl }}/images/codebreaker_2018/Task_1/ransom_note.png)

These values can be located in pcap for reference

![_config.yml]({{ site.baseurl }}/images/codebreaker_2018/Task_1/pcap.png)

The code needs to be followed to determine the other values. To narrow our focus on the interesting functions, we can look for the send function and then trace backwards. 

![_config.yml]({{ site.baseurl }}/images/codebreaker_2018/Task_1/send_function.png)

There are two calls to the function that sends data out, highlighted blue in the below picture. 

![_config.yml]({{ site.baseurl }}/images/codebreaker_2018/Task_1/comms_flow.png)

The call on the left of the graph is preceded by a call to recv, so this is not the call we are looking for. Following the program flow backwards, there are calls to _cid, _enc_ki, _c_hh and a few other functions. Using the names we can infer what the functions will most likely return. The function _enc_ki is the function used to encrypt the encryption key, _cid creates the victim id, _c_hh will create a message authentication code (MAC) using HMAC-SHA256 for the message being sent.<br>

- enc_ki likely stands for encrypt key or encrypt kit<br>
- cid likely stands for create ID or client ID<br>
- c_hh likely stands for content hmac helper<br>

![_config.yml]({{ site.baseurl }}/images/codebreaker_2018/Task_1/enc_ki.png)

The encrypted key is 0x200 chars. 

![_config.yml]({{ site.baseurl }}/images/codebreaker_2018/Task_1/cid_call.png)

The _cid function takes 3 arguments, each are pointers to data return, they are set to the IP address, victim id, and the time-based one time password (TOTP). 
(Correction: The 1st argument to _cid is a pointer to a buffer that should already be set to the IP address. If it is NULL then _cid will call the _gia function to determine the IP address.)

![_config.yml]({{ site.baseurl }}/images/codebreaker_2018/Task_1/otp_code.png)

![_config.yml]({{ site.baseurl }}/images/codebreaker_2018/Task_1/victim_id_code.png)

In the _cid function there is a call to get_totp_token, and then snprinf is used to convert the TOTP into a 6 digit string. The TOTP and the return from _gia are passed to the HMAC-SHA256, this is the victim id. The victim id and TOTP are concatenated and returned by the _cid, meaning the TOPT should be the 6 chars right after the victim id in the network traffic. 

![_config.yml]({{ site.baseurl }}/images/codebreaker_2018/Task_1/c_hh.png)

# Exfil Data #

The data sent by the ransomware to the attacker's server is in hex strings, so 0x2343 is "2343". The byte before the IP address, 02, is the message type and corresponds with the switch case number. I believe this is how the attacker knows how to process the message, since there can be any value for each of the different fields and the server can receive messages from multiple victims at a time. The full message is passed to _c_hh to create a message authentication code (MAC) of the message. 

Below is the lay out of the exfil data. 

![_config.yml]({{ site.baseurl }}/images/codebreaker_2018/Task_1/exfil_data.png)

Below is an example of the second message sent by the ransomware, message type 03. 

![_config.yml]({{ site.baseurl }}/images/codebreaker_2018/Task_1/exfil_data_2.png)

# Values in Ransom Storage #

As an alternative, the encryption key and TOTP can be found by examining the block chain, since all the tasks and files are open to us from the beginning. Below is an example of viewing some of the storage of the Ransom contract; details on retrieve each type of variable is provided in task 4. 

![_config.yml]({{ site.baseurl }}/images/codebreaker_2018/Task_1/ransom_storage.png)
