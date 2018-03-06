---
layout: post
title: SLAE Overview
subtitle: Starting the SLAE course
tags: [SLAE]
---

This is my first post on this page, and is the reason it was set up in the first place. Recently, I enrolled for the SecurityTube Linux Assembly Expert (SLAE) course to keep myself learning after finsihing my OSCP at the end of 2017. The end goal was to use this as a stepping stone to eventually go on to do the next level up from the OSCP - the OSCE.

So far I have completed the course material for the SLAE and am working my way through the questions. I plan to update this blog as I complete and have written up each question. The questions posed all seem really interesting, and am looking forward to learning more from this course. The requirements for the exam are:

**001 Create a Shell_Bind_TCP Shellcode**
	- Binds to a port
	- Execs a shell on incoming connection
	- Port number should be easily configurable

**002 Create a Shell_Reverse_TCP shellcode**
	- Reverse connects to configured IP and Port
	- Execs shell on succesful connection
	- IP and Port should be easily configurable

**003 Study the Egg Hunter Shellcode**
	- Create a working demo of the Egghunter
	- Should be configurable for different payloads

**004 Create a custom encoding scheme like the insertion encoder**
	- PoC using execve-stack as the shellcode to encode and execute

**005 Choose 3 shellcodes from msfvenom for linux/x86**
	- Used GDB/Ndisasm/Libemu to dissect the functionality of the shell code
	- Present the analysis

**006 Choose 3 shell codes from shell-storm and create polymorphic versions of them to beat pattern matching**
	- Cannot be larger than 150% of the shellcode
	- Bonus for it being smaller
	
**007 Create a custom crypter** 
	- Free to use any existing encryption schema
	- Can use any language

[SLAE Course] (http://www.securitytube-training.com/online-courses/securitytube-linux-assembly-expert/index.html)
