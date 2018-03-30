---
layout: post
title: SLAE Exercise 6 - Polymorphic Shellcode Part 1 Stime
subtitle: Polymorphic Shellcode Part 1
tags: [SLAE]
---

Overview
======

The first chosen shellcode from shell-storm is the 'stime and exit' shellcode. This can be found ![here](http://shell-storm.org/shellcode/files/shellcode-213.php). This shellcode sets the device's time to 0, and then gracefully exits. 

The original code can be seen fully commented below:




Requirements
------
