---
layout: default
permalink: /CodeBreaker-2019-Task-2/
title: NSA Codebreaker 2019, Task 2
---

Task 2 requires you to examine the android app TerrorTime to extract basic metadata. To complete this task you need to submit<br>
- the APK's permissions, <br>
- the SHA256 of the code signing certificate, and <br>
- the common name of the certificate signer. 

I used the tool jadx to read TerrorTime's AndroidManifest.xml file to extract the permissions. 

![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/task_2/permissions.png)

It took me some time to extract the signing certificate. A Google search will provide instructions to extract the certificate using tools such as unzip, jarsigner or apktool. These methods are for [APK Signature Scheme v1](https://source.android.com/security/apksigning#v1) which is based on signed JAR. TerrorTime was signed using [APK Signature Scheme v2](https://source.android.com/security/apksigning/v2) which inserts the certificate into the ZIP file before the central directory. This enables verification of the ZIP contents and metadata. 

![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/task_2/signing_schemes.png)
Image from [Android Source](https://source.android.com/security/apksigning/v2)

If the APK is decompressed using a tool like unzip, the certificate data is ignored and it appears as if the app was not signed. Below is the output of jarsigner. 
![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/task_2/unsigned_jar.png)

The tool jadx can extract and parse the APK Signature Scheme v2 certificate. 

![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/task_2/signature.png)