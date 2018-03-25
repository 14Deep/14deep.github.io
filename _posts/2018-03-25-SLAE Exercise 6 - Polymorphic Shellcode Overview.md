---
layout: post
title: SLAE Exercise 6 - Polymorphic Shellcode Overview
subtitle: Polymorphic Shellcode Overview
tags: [SLAE]
---

Overview
======

The fifth task of the SLAE exam requires 3 shellcodes to be chosen from [shell-storm](http://shell-storm.org/shellcode/) and polymorphic versions of each of them created to beat pattern matching. The requirements for this were as follows:

- Cannot be larger than 150% of the shellcode
- Bonus for it being smaller

Requirements
------

To determine how to carry out this exercise it needs to be understood what is meant by polymorphic shellcode, and how this can be applied to openly available shellcode. Polymorphic code in general is defined as code that mutates, is different to the original code however keeps the original function the same. Essentially code that is written differently but carries out the same functionality. This is done in an attempt to bypass security tooling by altering the signature from the original code. 

This can be achieved in many different ways, such as using different registers, using different instructions that will carry out the same function, change the order of the code and insert unnecessary code that doesn't alter the program.  There are many more ways that this can be achieved, not just by the ways listed above.

It has to be kept in mind however that shellcode is best the smaller it is, so where possible the size of the shellcode should not be increased too much. This is ultimately dependant on the amount of space available to insert the shellcode. 
