---
layout: post
title: SLAE Exercise 5 - Overview
subtitle: Shellcode Analysis
tags: [SLAE]
---

**Overview**

The fifth task of the SLAE exam requires a bind shell to be written. The requirements for this were as follows:

- Used GDB/Ndisasm/Libemu to dissect the functionality of the shell code
- Present the analysis

**Requirements**

For this exercise it was required to present full analysis on 3 different shellcodes found within metasploit's msfvenom. This tool is used to generate payloads depending on the attacker's requirements. Before the shellcode can be analysed, they have to be created.  The payloads for x86 Linux systems can be observed within msfvenom using the following command:

 \* msfvenom -l payload | grep linux/x86 \*

With the available payloads able to be seen below:

![Payloads](https://raw.githubusercontent.com/14Deep/14deep.github.io/master/_posts/Images/EX5/Payloads.png)


Once the payload required has been chosen, it can be created and saved using the following command:

 \* msfvenom -p linux/x86/payload1 -f raw > payload1 \*
	


**Analysing the shellcode**

Two methods will be used to analyse the shellcode - GDB and Ndisasm. Ndiasm allows the created payload to be disassembled and the instructions observed. GDB allows the shellcode to be debugged with each of the instructions being stepped through. Ndisasm will be used first review each line of the code and what it is doing, with GDB being used afterwards to step through these instructions and see the key activity occurring. 

**Ndiasm** 

Ndiasm (Netwide Disassembler) is simply just a disassembler. It can be used to disassemble binary files allowing analysis to be carried out on the disassembled file. This program can be used with the following format:

 \* ndisasm -b {16|32|64} filename \*
	
The binaries that will be looked at for this exercise the '32' parameter will be used as would be expected. 

**GDB**

GDB (GNU Project Debugger) is a debugger that allows the execution of a program from the inside, to observe each step that it takes. The output of using this will be explained when stepping through each of the selected shellcodes.  
