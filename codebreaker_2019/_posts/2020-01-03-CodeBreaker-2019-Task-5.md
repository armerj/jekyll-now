---
layout: default
permalink: /CodeBreaker-2019-Task-5/
title: NSA Codebreaker 2019, Task 5
---

Task 5 requires you to determine the organization's leader and submit his last encrypted message. 


*https://gist.github.com/nstarke/615ca3603fdded8aee47fab6f4917826*
*https://developer.android.com/studio/debug*
https://securitygrind.com/how-to-exploit-a-debuggable-android-application/


# Analyzing the OAUTH Verification Python Script #
The task provides a compiled python script that the server uses to verify a user's OAUTH tokens when logging into the XMPP server. I ran *file* on the pyc file which reported that it was a "python 2.7 byte-compiled" file. I used *uncompyle6* to decompile the file to a python script. 

{% highlight python %}
...
def check_token(token, url):
    try:
        data = 'token=' + token
        req = urllib2.Request(url, data=data)
        resp = urllib2.urlopen(req)
    except:
        logger.exception('Token introspect request failed')
        return False
    else:
        if resp.getcode() != 200:
            logger.error('Bad HTTP response code: %d', resp.getcode())
            return False
        try:
            j = json.loads(resp.read())
        except:
            logger.exception('Malformed response data')
            return False

        logger.debug('Token Verification Response: %s', pprint.pformat(j))
        try:
            if j[u'active'] and u'chat' in j[u'scope'] and j[u'token_type'] == u'access_token':
                return True
        except:
            logger.exception('Exception in token response logic')
            return False

    return False
...
{% endhighlight %}

The file contains two functions, main and check_token. The important function is check_token, which<br>
- makes a request to a oauth2 server to introspect the token and<br>
- checks if the token is
	- active, 
	- scope contains "chat", and
	- the token type is "access_token".

The token is comprised of the clientID and a value derived from the client's secert. 

{% highlight python %}
from base64 import b64decode
b64decode("a2luZ3NsZXktLXZob3N0LTE3NEB0ZXJyb3J0aW1lLmFwcDp6ZENVQ2RubUVLdDB0Ug==") # user's token

# outputs 'kingsley--vhost-174@terrortime.app:zdCUCdnmEKt0tR'
{% endhighlight %}

I didn't see any method to get a token for a different user. Then, I realized that the token is never checked against the user logging in. Remeber from task 3 that the database contains an entry for clientID and XMPP name. The token is only based on the clientID and the XMPP name is used to log into the XMPP server. This means that I could leave the clientID unchanged but change the XMPP name to the user I wanted to masquerade as. 

The app would<br>
- Request a OAUTH token for Kingsley<br>
- Request to login to the XMPP server as Arianna with OAUTH token for Kingsley<br>

The server would<br>
- Check if the OAUTH token was vaild and for chat
- Allow login 

Using Logcat
Logcat can pull logs from an android app and print them to the console. The flag pid will limit the output to the app running under that pid and "*:V" means to print all tags verbose. 
{% highlight bash %}
.\adb.exe -e logcat --pid=21576 --format=color *:V
{% endhighlight %}

Authentication is not same as authorization. [Anum Siddiqui](https://medium.com/datadriveninvestor/authentication-vs-authorization-716fea914d55) said it well on his blog, "... authentication is the process of verifying oneself, while authorization is the process of verifying what you have access to." The server only checks for authentication and not authorization, enabling me to authenticate as Kingsley but login as Arianna. 

I can now masquerade as different users. I logged in as each user to determine the each user's contacts and map their relationships. 

![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/Task_5/user_connections.png)

The users are separated into clear groups. Arianna talks to Brian, who only talks with two other users. Zachary and Aliah each have their own groups that are more connected. This implies that Brian is the leader and Arianna, Zachary, and Aliah are cell leaders. 

![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/Task_5/user_connections_marked.png)

# Getting Encrypted Messages #

The messages that are not decrypted are silently dropped and not displayed to the user. 

![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/Task_5/logcat_decrypt.png)

There are a few methods I tried to get the messages with varying levels of success. 

## Burp Suite ##
I have used Burp Suite in the past to inspect HTTP(S) traffic and decided try it since the traffic was over port 443. To use Burp Suite a root certificate needs to be added to the android phone. This enables the android phone to trust certificates signed by Burp Suite. The following steps will install the root certificate<br>
- *X* *Though you may not have had to do that. Need to test if it did cert verification*

*set up proxy on android*

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

![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/Task_5/edit_host_file.png)

socat commands
{% highlight bash %}
# in one terminal
socat -v openssl-listen:443,cert=cert.pem,verify=0,reuseaddr,fork tcp4:localhost:6500
# in another terminal
socat -v tcp4-listen:6500,reuseaddr,fork ssl:chat.terrortime.app:443,verify=0
{% endhighlight %}

Unfortunately, the XMPP protocol uses STARTTLS (described in [RFC6120 5.4.2](https://www.ietf.org/rfc/rfc6120.txt)) to upgrade a normal connection to a secure connection. 

![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/Task_5/starttls_flow.png)

Socat does not handle STARTTLS, but instead expects the connection to start out as a secure connection. I looked at using ![striptls](https://github.com/tintinweb/striptls) to stop the upgrade to a TLS connection. This is in reference to CVE-2016-10027, which is a vulnerability in the Smack XMPP library. This tool did not work since the XMPP server is configured to require a TLS connection. 

![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/Task_5/starttls.png)

## Frida ##
![Frida](https://frida.re/docs/android/) can inject Javascript into a running Android app. It enables a user to hook or modify existing functions, along with running new code. Frida requires root access to the Android to run the frida server. If you do not have root access but the app is marked as debuggable, then [frida can be loaded](https://koz.io/library-injection-for-debuggable-android-apps/) using the debugger. Additionally, I used [11x256's blog](https://11x256.github.io/Frida-hooking-android-part-1/) to setup the python script to interact with Frida and the Javascript to inject. 

![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/Task_5/message_flow.png)

Python code<br>
{% highlight python %}
#from https://11x256.github.io/Frida-hooking-android-part-1/
import frida
import time
device = frida.get_usb_device()
pid = device.spawn(["com.example.a11x256.frida_test"])
device.resume(pid)
time.sleep(1) #Without it Java.perform silently fails
session = device.attach(pid)
script = session.create_script(open("hook_message.js").read())
script.load()

#prevent the python script from terminating
raw_input()
{% endhighlight %}

Below is the Javascript injected into the TerrorTime app; it hooks the decryptMessage and sendStanzaInternal functions.<br>
{% highlight javascript %}
function bin2string(array){ // https://gist.github.com/taterbase/2784890
	var result = "";
	for(var i = 0; i < array.length; ++i){
		result+= (String.fromCharCode(array[i]));
	}
	return result;
}

console.log("Script loaded successfully ");
Java.perform(function x(){ //Silently fails without the sleep from the python code
    console.log("Inside java perform function");
    //get a wrapper for our class
    var my_class = Java.use("com.badguy.terrortime.Client");
    
    my_class.decryptMessage.overload("com.badguy.terrortime.Message").implementation = function(y){
        console.log("Message:\n\tDate: " + " " + y.getCreationDate() + "\n\tFrom: " + y.getContactId() + "\n\tFrom client: " + y.isFromClient() +  "\n\tContent: " + bin2string(y.getContent()));
        
        var x = this.decryptMessage(y); // may need to cast this
        //console.log("Message:\n\tDate: " + x.getCreatedAt() + "\n\tFrom: " + x.getContactId() + "\n\tFrom client: " + x.isFromClient() +  "\n\tContent: " + bin2string(x.getContent()));
        return x
    };

    var my_class_2 = Java.use("org.jivesoftware.smack.tcp.XMPPTCPConnection");
    
    my_class_2.sendStanzaInternal.implementation = function(x){
        console.log("Sent Stanza:" + x.toString()); // Base class for XMPP Stanzas, which are called Stanza(/Packet) in older versions of Smack (i.e. < 4.1).
        // http://javadox.com/org.igniterealtime.smack/smack-core/4.1.1/org/jivesoftware/smack/packet/Stanza.html
        this.sendStanzaInternal(x);
    };
    console.log("Created hooks");
});
{% endhighlight %}

![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/Task_5/message_flow_modified.png)

By hooking the decryptMessage function, I was able to print out each message the TerrorTime app recieves. 

![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/Task_5/hook_message.png)



get TLS master secert

stunnel
https://security.stackexchange.com/questions/33374/whats-an-easy-way-to-perform-a-man-in-the-middle-attack-on-ssl
https://www.stunnel.org/
https://gist.github.com/ohpe/e02596a2c2247ea1a212e019c355e2c3


Things that should work, but I didn't try <br>
- xmpp man-in-the-middle,<br>
- [NoPE Proxy](https://github.com/summitt/Burp-Non-HTTP-Extension) (Burp-Non-HTTP-Extension),<br>
- building/modifying android app, and<br>
- using xmpp python library. 

I was now able to login as Brian and extract his last encrypted message to fulfill the task. 