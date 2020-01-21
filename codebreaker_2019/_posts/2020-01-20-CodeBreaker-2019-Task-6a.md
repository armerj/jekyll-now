---
layout: default
permalink: /CodeBreaker-2019-Task-6a/
title: NSA Codebreaker 2019, Task 6a
---

Task 6a requires you to<br>
- spoof a message and<br>
- prevent spoofed user from reading replies.<br>

When sending a message, TerrorTime will encrypt the message content with a symmetric key. This key is then encrypted using the sender's local public keys and the receiver's public keys. It's interesting that only the local public keys are used, and not all of the ones listed on the sender's VCard. The app makes the assumption that if the key is on the user's VCard, then it is also in the user's local copy. This prevents the message key from being encrypted with the spoof user's public key. I could send a message from a spoofed user and verify that it was using my key. When the spoofed user logged in, TerrorTime would not be able to decrypt the message and would ignore the message. 

Below is the format of the message body. 
<pre>
{ 
   "messageKey": { 
      "&lt;base64 encoded public key fingerprint&gt;" : 
           "&lt;base64 encoded public key&gt;",
      "&lt;base64 encoded public key fingerprint&gt;" : 
           "&lt;base64 encoded public key fingerprint&gt;"
   }, 
   "messageSig" : 
       "&lt;base64 encoded message signature&gt;"
   ,
   "message" : { 
       "msg" : 
           "&lt;base64 encoded encrypted message&gt;", 
       "iv" : 
           "&lt;base64 encoded iv&gt;"
   }
}
</pre>

Unfortunately, the recipient was not able to read the message either. I needed to looker deeper into how the message is decrypted. 

In Messaging.decryptMessage, after the message is decrypted an internal structure is checked. 

![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/task_6/check_internal_structure.png)

Below is the JSON of the internal structure. 
<pre>
{
   "&lt;client username&gt;":
      ["&lt;base64 encoded public key fingerprint&gt;"]
   ,
   "&lt;contact username&gt;":
      ["&lt;base64 encoded public key fingerprint&gt;"]
   ,
   "body":
      "&lt;Message between users&gt;"
}
</pre>

Example
<pre>
{
   "kingsley--vhost-174@terrortime.app":
      ["Icaj54y\/0uKYGaOA1dn9bO9whI0F8pvwJiX0K4efK2k=","EWiwDLmEZnoMQsivUpWpmoo55z1VEbQVHXkLD9msnW0="]
   ,
   "arianna--vhost-174@terrortime.app":
      ["\/f58KjUf4i+3lquXE6eMhJNVxNswU72bgKK6spBF1hw=","Icaj54y\/0uKYGaOA1dn9bO9whI0F8pvwJiX0K4efK2k=","EWiwDLmEZnoMQsivUpWpmoo55z1VEbQVHXkLD9msnW0="]
   ,
   "body":
      "This is my test for future messages"
}
</pre>


The each user's public key fingerprints in the internal structure are checked to determine if the the message key was encrypted with a corresponding public key. If the internal structure contains a fingerprint that was not used, then TerrorTime treats the message as corrupt and drops it. Interestingly, the app does not check if the public key belongs to the sender or the receiver. Additionally, the message key can be encrypted with a public key that is not recorded in the internal structure. This is why I could send messages without error. 

Below if the JavaScript I injected into TerrorTime to manipulate the message's internal structure to fix the corrupt issue. The script hooks the Messaging.encryptMessage function and changes the arguments sent to the real function. The client's public key fingerprints list is replaced with the contact's. Now when the contact receives the message, the internal structure only contains fingerprints for their own public keys.

![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/task_6/encryptMessage.png)

Removing the spoofed user's public key from their VCard is another method to achieve the same result. The second part of the task can be completed two different ways; only one is checked for by the scoring system. The first method is to add the recipient to the spoofed user's block list. This is not a function provided by the TerrorTime app, but is a function of the XMPP server. The second method is to remove the spoofed user's public key from their VCard. Their public key is added back to the server when the user next logs in, which prevents stopping communication permanently. 

In my opinion, adding the user to a block list is a better method, since it<br>
- has a more grainier focus,<br>
- enables spoofing while the user is logged in, and<br>
- is less detectable.<br>

Removing the public key has the following downsides,<br>
- blocks all contacts,<br>
- you are unable to control when spoofed user logs back in, and<br>
- is more detectable.<br>

Below is the JavaScript I injected to add user to a block list. I had some trouble accessing the Enum PrivacyItem.Type from the JavaScript. This forced me to hook the toXML function to set the privacyItem values. When a stanza is sent to the XMPP server, the send function will call the function toXML of the object representing the stanza. By hooking this function, I was able to format the XML correctly to block a user. 

![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/task_6/block_list.png)

Below is the XML representing a PrivacyItem. <br>
<pre>
&lt;item
	action="deny" 
	type="jid" 
	value="zachary--vhost-174@terrortime.app" 
	order="0"&gt;
	&lt;message/&gt;
&lt;/item&gt;
</pre>

Below are picture of the initial and final states of testing the method, along with a gif showing the message being blocked. The Powershell window is running an interactive Python session I used to interactive with Frida. Below are the steps I took:
- first sent a message to ensure connectivity,<br>
- added Zachary to Brian's block list,<br>
- sent a second message which was blocked,<br>
- added Zachary to Brian's allow list, and<br>
- sent a final message that was received. 
![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/task_6/privacy_block_intial.png)

Full video can be found [here](https://youtu.be/nEI1I4CRf0g). 
![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/task_6/block_list_fast.gif)


![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/task_6/privacy_block_final.png.png)

# Removing the public key from VCard #
I used the VCardHelper.savePublicKey function as a template to write JavaScript to replace the spoofed user's public key from their VCard with my own. The below JavaScript will monitor TerrorTime for instances of the class com.badguy.terrortime.TerrorTimeApplication. Once found, the javascript<br>
- retrieves the current instances of VCardManager and XMPPTCPConnection,<br>
- requests the current user's VCard,<br>
- sets the "DESC" to my public key,<br>
- saves the modified VCard.<br>
![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/task_6/replace_pubkey.png)

Now when a user replies to the spoofed user, the message key is not encrypted with their public key; preventing decryption. Removing the spoofed user's public key from their VCard fixes both problems, prevents the public key fingerprint from being in the messages internal structure and prevents the spoofed user from reading replies. 