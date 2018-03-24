---
layout: post
title: SLAE Exercise 5 (part 1) - Chmod
subtitle: Shellcode Analysis 1 - Chmod
tags: [SLAE]
---

linux/x86/chmod
======

The first chosen shellcode to be used from msfvenom is the 'chmod' payload. This, as expected, changes a specified files permissions. The parameters to be used for this payload can be seen with the following command:

	msfvenom -p linux/x86/chmod --payload-options
	
From this it can be seen that there are two required payloads:

