---
layout: default
permalink: /CodeBreaker-2019-Task-4/
title: NSA Codebreaker 2019, Task 4
---

Task 4 requires you to determine the pin to unlock the cached credentials in the recovered database. Originally, this task also required you to determine the action planned by analyzing their code words. This requirement was later removed. To complete this task you need to submit<br>
- The terrorist cell leader's username and <br>
- the date the cell is planning an action. <br>

The first step is to analyze the TerrorTime app to determine how the user logs in. 


** How user logs in **

First need to figure out how app uses pin

Find Activity

Check for call

Find call

Check how pin is checked
* How pin is stored *

The SHA256 hash of the pin is stored in the client SQLite database. I wrote a python script to brute force the pin. 
{% highlight python %}
from hashlib import sha256
hash_pin = "\xa9\x9d\x19\x16\x2a\xaa\xa4\x0a\xfa\xe0\xd6\xe3\xd0\x08\x40\x56\xce\xc1\x4a\x11\x54\x75\xb5\xxf7\x1f\x7b\xc1\x3e\xc9\xd8\x99\x8a"

for pin in range(0, 1000000):
    if hash_pin == sha256(str(pin)).digest():
        print pin

# output 123793
{% endhighlight %}

I used android studio to emulate a Nexus S android phone running Oreo. I used the Android Virtual Device (AVD) manager under Tools to create the device. To create a device <br>
- Click "Create Virtual Device",
- Choose type of device,
- Click Next,
- Choose system image,
- Give the device a name, and
- Click Finish. 

![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/task_4/avd.png)

The AVD manager or command line tool emulator.exe can start the emulator. Installing or copying a file is as simple as dragging the file onto the phone screen. I used adb.exe to interact with the phone file system. When I dragged the database file to the phone, it was downloaded to /sdcard/Downloads. 

** layout of phone data **
I used adb.exe to move the file from the Downloads folder to /data/user/0/com.badguy.terrortime/database/clientDB.db. 
Commands to connect adb.exe to android
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

[Connection error]
![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/task_4/conn_error.png)

When the codebreaker creators started designed the challenge they set the app to wait for 5 secs before timing out. This was not enough time due to the amount of participants. They released an updated version that waited 30 secs. 

I launched the TerrorTime and input the pin, enabling me to impersonate the terrorist, Kingsley. Kingsley has two contacts
- Arianna and 
- Devora.

Clicking on each contact's name retrieves their past messages. 

I figured out that Arianna is the cell leader due to the following messages
From|To|Message|Reason
----|---|------|------
Arianna|Kingsley|Is everything set?|She's checking on the status of the op
Arianna|Kingsley|Be grateful for your role in this, Kingsley|She's telling him to be grateful
Kingsley|Arianna|yes ma'am|He is very formal with her

![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/task_4/chat_1.png)

From|To|Message|Reason
----|---|------|------
Kingsley|Devora|want to make sure the ma'am is pleased and confident in us.|Wants to impress Arianna
Kingsley|Devora|see you after New Years Day|Is meeting Devora for the op
** need time message ** 

![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/task_4/chat_2.png)

I used the following messages to determine the date. 
From|To|Message|Reason
----|---|------|------
Arianna|Kingsley|Exactly. after the holiday, 1 day|She's telling him when the op is to happen
Kingsley|Devora|see you after New Years Day|Is meeting Devora for the op

I now knew the date of the upcoming operation, 01/02/2020. I used an online epoch converter to convert the date to epoch. 
01/02/2020 15:23 Z == 1577978580

determining action. I was only able to determine that wedding usually refers to a big attack (source). 

![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/task_3/dig_command.png)



