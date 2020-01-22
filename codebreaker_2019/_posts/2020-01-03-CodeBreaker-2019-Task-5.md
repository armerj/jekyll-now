---
layout: default
permalink: /CodeBreaker-2019-Task-5/
title: NSA Codebreaker 2019, Task 5
---

Task 5 requires you to determine the organization's leader's identity and submit his last encrypted message. 

# Analyzing the OAUTH Verification Python Script #
The task provides a compiled python script that the server uses to verify a user's OAUTH tokens when logging into the XMPP server. I ran *file* on the pyc file which reported that it was a "python 2.7 byte-compiled" file. I used *uncompyle6* to decompile the file to a python script. 

![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/task_5/auth_py.png)

The file contains two functions, main and check_token. The important function is check_token, which<br>
- makes a request to a oauth2 server to introspect the token and<br>
- checks if the token is
	- active, 
	- scope contains "chat", and
	- the token type is "access_token".

The token is comprised of the clientID and a value derived from the client's secret. 

{% highlight python %}
from base64 import b64decode
b64decode("a2luZ3NsZXktLXZob3N0LTE3NEB0ZXJyb3J0aW1lLmFwcDp6ZENVQ2RubUVLdDB0Ug==") # user's token

# outputs 'kingsley--vhost-174@terrortime.app:zdCUCdnmEKt0tR'
{% endhighlight %}

I didn't see any method to get a token for a different user. Then, I realized that the token is never checked against the user logging in. Remember from task 3 that the database contains an entry for clientID and XMPP name. The token is only based on the clientID and the XMPP name is used to log into the XMPP server. This means that I could leave the clientID unchanged but change the XMPP name to the user I wanted to masquerade as. 

The app would<br>
- Request a OAUTH token for Kingsley<br>
- Request to login to the XMPP server as Arianna with OAUTH token for Kingsley<br>

The server would<br>
- Check if the OAUTH token was vaild and for chat
- Allow login 

Logcat can pull logs from an android app and print them to the console. The flag pid will limit the output to the app running under that pid and "*:V" means to print all tags with verbose output. 
{% highlight bash %}
.\adb.exe -e logcat --pid=21576 --format=color *:V
{% endhighlight %}

Authentication is not same as authorization. [Anum Siddiqui](https://medium.com/datadriveninvestor/authentication-vs-authorization-716fea914d55) said it well on his blog, "... authentication is the process of verifying oneself, while authorization is the process of verifying what you have access to." The server only checks for authentication and not authorization, enabling me to authenticate as Kingsley but login as Arianna. 

I can now masquerade as different users. I logged in as each user to determine each user's contacts and map their relationships. 

![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/task_5/user_connections.png)

The users are separated into clear groups. Arianna talks to Brian, who only talks with two other users. Zachary and Aliah each have their own groups that are more connected. This implies that Brian is the leader and Arianna, Zachary, and Aliah are cell leaders. 

![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/task_5/user_connections_marked.png)

# Getting Encrypted Messages #

The messages that are not decrypted are silently dropped and not displayed to the user. 

![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/task_5/logcat_decrypt.png)

There are a few methods I tried to get the messages with varying levels of success. 

## Burp Suite ##
I have used Burp Suite in the past to inspect HTTP(S) traffic and decided to try it since the traffic was over port 443. Normally, to use Burp Suite a root certificate needs to be added to the android phone. This enables the android phone to trust certificates signed by Burp Suite. The following steps, from [Distributed Compute](https://distributedcompute.com/2019/08/15/tech-note-installing-burp-certificate-on-android-9/), will install the root certificate.<br>
- Export Burp CA certificate from Proxy -> Options page in DER format<br>
- Convert DER to PEM format and rename as subject hash<br>
{% highlight bash %}
openssl x509 -inform DER -in burp.der -out burp.pem
openssl x509 -inform PEM -subject_hash_old -in burp.pem | head -1
mv burp.pem <previous command output>
{% endhighlight %}
- Copy certificate to Android 
- Restart the emulator with system writeable option and remount /system as read/write
{% highlight bash %}
emulator.exe -avd <avd_name> -writable-system
adb shell su -c “mount -o rw,remount,rw /”
adb shell
{% endhighlight %}
- Move file to trusted certificate store
{% highlight bash %}
cp /sdcard/Downloads/<subject_hash>.0 /system/etc/security/cacerts
chmod 644 /system/etc/security/cacerts/<subject_hash>.0
reboot
{% endhighlight %}

This step was not required since TerrorTime does not check that the certificate is chained to a trusted CA. 

The Android's proxy configuration is under Extended Controls -> Settings -> Proxy. 
![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/task_5/proxy.png)

This setup does not work since Burp only handles HTTP(S) traffic, even though the XMPP traffic is using port 443 which is for secure web traffic. Unfortunately, Burp ignores it since its not HTTP or HTTPS traffic. I did not try it, but there is an extension called [NoPE Proxy](https://github.com/summitt/Burp-Non-HTTP-Extension) that handles non-HTTP traffic. 

## Socat ##
Socat is a tool that can redirect and manipulate network traffic. I followed a guide by [PenTestPartners](https://www.pentestpartners.com/security-blog/socat-fu-lesson/) to setup socat to <br>
- listen on one port,<br>
- strip the SSL,<br>
- send traffic to another port,<br>
- add SSL back and forward to server.<br>

This would enable me to run Wireshark and capture the traffic while it is in plaintext. 

There are two methods to redirect traffic to the socat listener<br>
- change the IP address returned by the DNS query, or<br>
- change the domain name in the database.<br>

I probably choose the harder method of changing the DNS response by editing the host file. 

![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/task_5/edit_host_file.png)

socat commands
{% highlight bash %}
# in one terminal
socat -v openssl-listen:443,cert=cert.pem,verify=0,reuseaddr,fork tcp4:localhost:6500
# in another terminal
socat -v tcp4-listen:6500,reuseaddr,fork ssl:chat.terrortime.app:443,verify=0
{% endhighlight %}

Unfortunately, the XMPP protocol uses STARTTLS (described in [RFC6120 5.4.2](https://www.ietf.org/rfc/rfc6120.txt)) to upgrade a normal connection to a secure connection. 

![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/task_5/starttls_flow.png)

Socat does not handle STARTTLS, but instead expects the connection to start out as a secure connection. I looked at using [striptls](https://github.com/tintinweb/striptls) to stop the upgrade to a TLS connection. This is in reference to CVE-2016-10027, which is a vulnerability in the Smack XMPP library. This tool did not work since the XMPP server is configured to require a TLS connection. 

![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/task_5/starttls.png)

## Frida ##
[Frida](https://frida.re/docs/android/) can inject JavaScript into a running Android app. It enables a user to hook or modify existing functions, along with running new code. Frida requires root access to the Android to run the Frida server. If you do not have root access but the app is marked as debuggable, then [frida can be loaded](https://koz.io/library-injection-for-debuggable-android-apps/) using the debugger. Additionally, I used [11x256's blog](https://11x256.github.io/Frida-hooking-android-part-1/) to setup the python script to interact with Frida and the JavaScript to inject. 

Below is the normal flow from logging in to the decryptMessage function. <br>
![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/task_5/message_flow.png)

Python code to start TerrorTime and inject JavaScript.
![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/task_5/python.png)

Below is the JavaScript injected into the TerrorTime app; it hooks the decryptMessage to print to the console all messages received. Additionally, it hooks functions called when sending and receiving Stanzas to print them to the console. <br>
![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/task_5/js.png)

New flow after hooking decryptMessage.<br>
![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/task_5/message_flow_modified.png)

By hooking the decryptMessage function, I was able to print out each message the TerrorTime app receives. 

![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/task_5/hook_message.png)

Things that should work, but I didn't try <br>
- xmpp man-in-the-middle,<br>
- [NoPE Proxy](https://github.com/summitt/Burp-Non-HTTP-Extension) (Burp-Non-HTTP-Extension),<br>
- building/modifying android app,<br>
- using xmpp python library,<br>
- stunnel, can handle STARTTLS, and<br>
- retrieving the TLS master secret from the Android app.<br>

I was now able to login as Brian and extract his last encrypted message to fulfill the task. 

[Back to Overview](https://armerj.github.io/CodeBreaker-2019-Overview/)