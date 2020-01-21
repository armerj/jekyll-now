---
layout: default
permalink: /CodeBreaker-2019-Task-6b/
title: NSA Codebreaker 2019, Task 6b
---

Task 6b requires you to determine a method to decrypt all future messages sent in TerrorTime. 

This is possible since a user's VCard can have an arbitrary number of public keys and each outgoing message is encrypted with all public keys. Cycling keys can help protect previous messages if a key is compromised in the future. TerrorTime's implication of this technique is very poor. It would have been better to limit the VCard to only having one public key. 

It should be noted, that if the public key is only updated for one user, the organization leader, it will render all future messages he sends unreadable by the recipient. As mentioned in task 6a, when a message is sent only the local public keys are used to encrypt the message body, but all public key fingerprints are added to the internal message structure. When TerrorTime decrypts the message the public key fingerprints are checked against all public key used to encrypt the message key. If there is a fingerprint in the internal message structure not used to encrypt the message key, then the message is reported as corrupted and dropped. The spoofed user's local copy would not contain our public key, resulting in the message key not being encrypted with this key and breaking future decryption of the message inside of the app. 

Uploading the key to every user's VCard side steps this problem. Now when a message is encrypted our public key is used to encrypt the message, since it is on the recipient's VCard. TerrorTime does not check which user the public key was from, but rather only if it was used. 

# Generating RSA Key #

I used openssl to generate a new RSA public and private key pair to upload to each user's VCard. 

{% highlight bash %}
openssl genpkey -algorithm rsa -out openssl_generated_pubkey.pem
openssl rsa -inform PEM -pubin -text -in openssl_generated_pubkey.pem
{% endhighlight %}

# Using Frida #

The below javascript will inject the public key stored in variable new_pub_key to the current user's VCard. It uses the functions defined in VCardHelper class to<br>
- log current VCard's public keys,<br>
- add new key, and<br>
- retrieves public keys to verified it was added.<br>

![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/task_6/save_key.png)

# Using Legitimate Functionality #

When a user logs into TerrorTime, a request is made to the user's VCard which contains the user's public keys. The app will update the VCard with any missing public keys from the local database. One method to add a public key to each user's VCard is to update the database with a new public and private key and masquerade as each user.

The private key is stored encrypted in the client database. I used Frida to retrieve the client's symmetric key and then used python to encrypt the key. One interesting thing I discovered is that the object needs to be cast as the parent object if you want to call functions from the parent object. For an example, to call the function getEncoded on the object returned from the function generateSymmetricKey, it must be cast to Key even though the object is of the type SecertKey. I believe this is because the class Key defines the getEncoded function. 

Javascript injected into process to get client's symmetric key. 
![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/task_6/clients_key.png)

Python script to encrypt with the client's symmetric key. The resulting key can be added to the client database along with the matching private key. 

{% highlight python %}
from Crypto.Cipher import AES
from binascii import hexlify, unhexlify
from base64 import b64decode

with open("brian_key.pem") as fi:
    priv_key_pem = fi.read() # .split("\n")[1:-1]

priv_key = priv_key_pem # b64decode("".join(priv_key_pem))

pad_amount = (16 - len(priv_key)) % 16
priv_key += pad_amount * chr(pad_amount)
symkey = b64decode("KcnLeCD2MQzIkGRq7Xmf0kdHnhOAEpVVvk6/HoWEs+w=")
enc_key = AES.new(symkey, AES.MODE_ECB).encrypt(priv_key)

with open("enc_brian_key_pem", "w") as fo:
    fo.write(enc_key)
{% endhighlight %}

I can now decrypt all future messages sent through TerrorTime. 