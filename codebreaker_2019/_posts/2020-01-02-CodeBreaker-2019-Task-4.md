---
layout: default
permalink: /CodeBreaker-2019-Task-4/
title: NSA Codebreaker 2019, Task 4
---

Task 4 requires you to determine the pin to unlock the cached credentials in the recovered database. Originally, this task also required you to determine the action planned by analyzing their code words. This requirement was later removed. To complete this task you need to submit<br>
- The terrorist cell leader's username and <br>
- the date the cell is planning an action. <br>

The first step is to analyze the TerrorTime app to determine how the user logs in. I used android studio to emulate a Nexus S android phone running Oreo. Android Virtual Device (AVD) manager, under Tools, can create the device. To create a device <br>
- click "Create Virtual Device",
- choose type of device,
- click Next,
- choose system image,
- give the device a name, and
- click Finish. 

![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/task_4/avd.png)

The AVD manager or command line tool emulator.exe can start the emulator. Installing or copying a file is as simple as dragging the file onto the phone screen. After installing TerrorTime, I opened it and tried to log in, which requires a username and a pin. I only had the username from the database but not the matching pin. I decided to follow the code that handles the login to determine how the pin is checked. The [Activity Class](https://developer.android.com/guide/components/activities/intro-activities) in an app handles the window the app draws it's UI in. TerrorTime contains six Activity Classes:<br>
- ChatActivity,<br>
- ContactActivity,<br>
- LoginActivity,<br>
- MainActivity,<br>
- RegisterActivity, and<br>
- SettingsActivity.<br> 

The LoginActivity handles the UI for the login screen and handles the login validation. The tool jadx was not able to correctly decompile the UserLoginTask Class so it dumps the instructions to the screen. 

![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/task_4/dump_inst.png)

Looking through the instructions, I saw calls to initialize the Client Class and then set encryptPin to the entered pin. 

![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/task_4/setEncrypt.png)

Further in LoginActivity is a function called doInBackground, which uses the created Client Class to generate a symmetric key and validate a access token. I determined this function to be interesting since it contained error messages referring to logging in, client ID, and pin. 

![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/task_4/background.png)

The Client.generateSymmetricKey function<br>
- retrieves the encryptPin,<br>
- SHA256 hashes the pin,<br>
- sets checkPin variable to the hash, and<br>
- returns a generated key using the hash and AES.<br> 

I remembered from task 2, that the database contains the client secret. The code indicates that the stored secret is the SHA256 hash of the pin. To determine the length of the field I checked the ParameterValidatorClass. The function isValidPin checks if the entered pin is six digits long. 

![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/task_4/validator.png)

The number of possible combinations for the pin is 10^6 or 1,000,000, which is simple to brute force. I wrote a python script to brute force the pin. <br>

{% highlight python %}
from hashlib import sha256
hash_pin = "\xa9\x9d\x19\x16\x2a\xaa\xa4\x0a\xfa\xe0\xd6\xe3\xd0\x08\x40\x56\xce\xc1\x4a\x11\x54\x75\xb5\xxf7\x1f\x7b\xc1\x3e\xc9\xd8\x99\x8a"

for pin in range(0, 1000000):
    if hash_pin == sha256(str(pin)).digest():
        print pin

# output 123793
{% endhighlight %}

I used adb.exe to interact with the phone file system. When I dragged the database file to the phone, it was downloaded to /sdcard/Downloads. 

I used adb.exe to move the file from the Downloads folder to /data/user/0/com.badguy.terrortime/database/clientDB.db.<br> 
Commands to connect adb.exe to Android.
{% highlight powershell %}
.\adb.exe root
.\adb.exe remount
.\adb.exe shell
{% endhighlight %}

Commands executed with adb.exe. 
{% highlight bash %}
cd /data/user/0/com.badguy.terrortime
ls -al
su <app user>
cp /sdcard/Downloads/clientDB.db database/clientDB.db
{% endhighlight %}

![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/task_4/conn_error.png)

When the codebreaker creators designed the challenge they set the app to wait for 5 secs before timing out. This was not enough time due to the amount of participants. They released an updated version that waited 30 secs. 

I launched the TerrorTime and input the pin, enabling me to impersonate the terrorist, Kingsley, who has two contacts,
- Arianna and 
- Devora.

Clicking on each contact's name retrieves their past messages. 

I figured out that Arianna is the cell leader due to the following messages
|From|To|Message|Reason|
|----|---|------|------|

|Arianna|Kingsley|Is everything set?|She's checking on the status of the op|
|Arianna|Kingsley|Be grateful for your role in this, Kingsley|She's telling him to be grateful|
|Kingsley|Arianna|yes ma'am|He is very formal with her|

![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/task_4/chat_1.png)

|From|To|Message|Reason|
|----|---|------|------|

|Kingsley|Devora|want to make sure the ma'am is pleased and confident in us.|Wants to impress Arianna|
|Kingsley|Devora|see you after New Years Day|Is meeting Devora for the op|

![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/task_4/chat_2.png)

I used the following messages to determine the date. 
|From|To|Message|Reason|
|----|---|------|------|

|Arianna|Kingsley|Exactly. after the holiday, 1 day|She's telling him when the op is to happen|
|Kingsley|Devora|see you after New Years Day|Is meeting Devora for the op|
|Kingsley|Devora|we're to acquire the engagement ring at 1523|Mentions a time for an action|

I now knew the date of the upcoming operation, 01/02/2020, and the time, 1523. I used an online epoch converter to convert the date to epoch. 
01/02/2020 15:23 Z == 1577978580

determining action. I was only able to determine that wedding usually refers to a big attack (source). 

[Back to Overview](https://armerj.github.io/CodeBreaker-2019-Overview/)


