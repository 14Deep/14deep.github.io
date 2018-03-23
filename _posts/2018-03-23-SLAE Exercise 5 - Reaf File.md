---
layout: post
title: SLAE Exercise 5 - Read File
subtitle: Shellcode Analysis 2 - Read File
tags: [SLAE]
---

**linux/x86/read_file**

The second chosen shellcode to be used from msfvenom is the 'read_file' payload. This, as expected, reads a specified file and displays it on the screen. The parameters to be used for this payload can be seen with the following command:

	msfvenom -p linux/x86/read_file --payload-options
	
The command that is used to create a payload that will read the '/etc/passwd' file is as below:

	msfvenom -p linux/x86/read_file PATH=/etc/passwd -f raw > readfileraw

Once this has been generated it can be disassembled using ndisasm to analyse the instructions used with the following command:

	ndisasm -b 32 ./readfile

![Payloads](https://raw.githubusercontent.com/14Deep/14deep.github.io/master/_posts/Images/EX5/part2/Ndisasm.png)

**Stepping through the instructions**

Each of the instructions seen here will be stepped through, describing what is happening at each step. 



**open**  int open(const char *pathname, int flags);

*jmp short 0x38*- jumps to location 0x38, which in this instance calls 0x2 (the next instruction). This is used as a 'jmp call pop' to push the next instruction after call onto the stack, to be used as the pathname pointer.

*mov eax, 0x5* - move 5 to eax, used for the open(2) syscall

*pop ebx* - pop value in stack to ebx which should contain the pathname. 

*xor ecx, ecx* - clear the ecx register 

int 0x80 - interrupt to call syscall


**read**  ssize_t read(int fd, void *buf, size_t count);

mov ebx, eax - move the return value into ebx (the fd)

mov eax, 0x3 - move 3 into eax for the read(2) syscall

mov edi, esp - move stack pointer to edi

mov ecx, edi - then move the value from the stack pointer into ecx for the syscall

mov edx, 0x1000 - move 0x1000 into edx, page file? 

int 0x80 - interrupt to call syscall

 
**write**  ssize_t write(int fd, const void *buf, size_t count);

mov edx, eax - move return value to edx (number of bytes read is returned to eax)

mov eax, 0x4 - move 4 to eax for write(2) syscall

mov ebx, 0x1 - move 1 to ebx

int 0x80 - interrupt to call syscall


**exit**

mov eax, 0x1 - move 1 to eax (exit syscall to exit gracefully)

mov ebx, 0x0 - move 0 to ebx

int 0x80 - interrupt to call syscall





