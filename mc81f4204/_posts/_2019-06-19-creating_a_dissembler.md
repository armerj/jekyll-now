---
layout: default
permalink: /creating-a-dissembler/
title: Creating a Dissembler
---

There are three important plugin types for adding a new architecture to radare2: asm, analysis, and bin. The documentation I could find for creating new plugins for radare2 referenced toy examples or fully complete plugins. At first I felt overwhelmed and had no idea how to bridge this gap. Thankfully, the plugin code is relatively small and was easy to understand after I began reading the already complete plugins. I referenced the plugins for COFF, 8051 and NES when creating my own plugin for the MC81F4204. 

Radare2 uses the asm plugin to compile and decompile code for an architecture. The examples I found contained an array describing each opcode and how it should be dissembled. The plugin would then loop through this array checking the current binary data to the available opcodes. 

In my opinion, the analysis plugin is the most important plugin of the three. This plugin allows radare2 to model the decompile code in a generic way to perform analysis. The plugin only needs to tell radare2 what type of instruction each opcode is, for example OR, CMP or CALL. After this plugin is created all analysis code radare2 has can be used to analyze the new architecture. This plugin can also define the ESIL ***what is esil*** for each instruction, allowing radare2 to emulate instructions. ESIL allows an analyst to determine indirect jumps or safely test unknown code through emulation. 

The bin plugin defines a file format

Steps for enabling each plugin type



Enabling for windows
https://github.com/armerj/radare2/pull/2/files



![_config.yml]({{ site.baseurl }}/images/Codebreaker/Task_1/complete.png)
