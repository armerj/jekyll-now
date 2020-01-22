---
layout: default
permalink: /CodeBreaker-2019-Task-7/
title: NSA Codebreaker 2019, Task 7
---

Task 7 requires you to<br>
- decrypt


Analyzing app
Determining it was rust
Checking libray on andriod phone, also rust
General things about rust (returns <error>,<return val>, return ptr is first arg, and not in eax. 

Different ways to backdoor rsa, papers
fixed primes
enc(p) in e, long e, e is usually 3 or X
enc(p) in n
They actually did key_2 ^ enc(key_1 ^ p) in n, made it so bits were not exact. 2048 bit key was 2046-2048

This is not the best scheme, since no matter what size the key pair generated, the secert master key is half that size. 


Finding what they did
Function wasn't hard to locate, named rsa alg1 or something
There was another rsa alg you could compare the two to see where back door code was
Saw func for selecting params, which used embedded xor keys and public key

easiest method for following code was through coloring blocks and looking for for loops, then determining what each block did. 
** Write down what each block did in a structure ** (for\ngenerate prime\nblah...) \
** Got through image showing each piece **
examples like bit size checking, encrypting p, div and check 
Looked at each code block to get a general idea what it was doing and then stepped up from there 

permute_key just hashes the r key. Interesting it has code for MD5 hash, which is 128 bits implying that the exe might be able to used to backdoor 256 bit keys, even though it asks for 512, 1024, 2048. Annoying rust use of vectors and slices. hash is put on stack using xmm to load as a slice or vector

Now we figure out what they did, but we don't have their private key... lucky for us they backdoor'd their backdoor. We can use the 256 -> 512 -> 1024 -> TerrorTime 2048 keys. There may have been another way to do it, but 256 bit is really low and there are tools out there to factor these size numbers. I used yafu, and it was done in a few mins. 

python code to get certs

pull all public keys

decrypt all public keys

code to pull messages

code to decrypt messages

the answers to the questions

Other ways
Dynamically

Ghidra

Other
Looking at distrubution of keys, graph fir X bits and see what it looks like
Check leading byte and see if 1 has a higher chance than others
Check key sizes, what do they normally land on 


![_config.yml]({{ site.baseurl }}/images/codebreaker_2019/Task_5/hook_message.png)