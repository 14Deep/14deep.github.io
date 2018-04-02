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

For this exercise it was required to present full analysis on 3 different shellcodes found within Metasploit's msfvenom. This tool is used to generate arbritary payloads. Before the shellcode can be analysed, they have to be created. The payloads for x86 Linux systems can be observed within msfvenom using the following command:
```
msfvenom -l payload | grep linux/x86
```
With the available payloads able to be seen below:

![Payloads](https://raw.githubusercontent.com/14Deep/14deep.github.io/master/_posts/Images/EX5/Payloads.png)

The chosen payloads will be different to ones looked at before within the course and the exam exercises. Once the payload required has been chosen, it can be created and saved using the following command:
```
msfvenom -p linux/x86/payload1 -f raw > payload1
```	
It is important to note that shellcodes may require other parameters when created, this depends on the shellcode used and will be looked at within the chosen shellcodes.

**Analysing the shellcode**

Two methods will be used to analyse the shellcode - GDB and Ndisasm. Ndiasm allows the created payload to be disassembled and the instructions observed. GDB allows the shellcode to be debugged with each of the instructions being stepped through. Ndisasm will be used first to review each line of the code and what it is doing, with GDB being used afterwards to step through these instructions and see the key activity identified from Ndisasm occurring. 

**Ndiasm** 

Ndiasm (Netwide Disassembler) is simply just a disassembler. It can be used to disassemble binary files allowing analysis to be carried out on the disassembled file. This program can be used with the following format:
```
ndisasm -b {16|32|64} filename 
```	
The binaries that will be looked at for this exercise the '32' parameter will be used as would be expected for 32 bit binaries. 

**GDB**

GDB (GNU Project Debugger) is a debugger that allows the execution of a program from the inside, to observe each step that it takes. The output of using this will be explained when stepping through each of the selected shellcodes.  


This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert Certification:

http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert

Student ID: 1092
