---
layout: post
title: SLAE Exercise 5 (part 1) - Chmod
subtitle: Shellcode Analysis 1 - Chmod
tags: [SLAE]
---

linux/x86/chmod
======

The first chosen shellcode to be used from msfvenom is the 'chmod' payload. This, as expected, changes a specified files permissions. The parameters to be used for this payload to determine what payloads are required can be seen with the following command:

	msfvenom -p linux/x86/chmod --payload-options
	
From this it can be seen that there are two required options:

![payloadoptions](https://raw.githubusercontent.com/14Deep/14deep.github.io/master/_posts/Images/EX5/part1/payloadoptions.png)

The command that is used to create the payload  is as below, for this no changes to the default options will be made:

	msfvenom -p linux/x86/chmod  -f raw > chmodraw

Once this has been generated it can be disassembled using ndisasm to analyse the instructions used with the following command:

	ndisasm -b 32 ./chmodraw

![ndisasm](https://raw.githubusercontent.com/14Deep/14deep.github.io/master/_posts/Images/EX5/part1/ndisasm.png)


Stepping through the instructions
------

Each of the instructions seen here will be stepped through, describing what is happening at each step. 

**int chmod(const char *pathname, mode_t mode);**


```
Cdq - Convert doubleword to a quadword, extending the sign bit of eax into the edx register. This should effectively clear the edx register.

push byte +0xf - Push 15 onto the stack

pop eax - Pop 15 into eax, this will be the syscall value 'sys_chmod'.

push edx - The value 0 is pushed onto the stack. 

call dword 0x16 - The instruction at 0x16 is called, in this instance it is 'pop ebx'


The following instructions are not used as instructions and infact represent the value moved to the where ebx will point to in the instruction that is called to.
Das	
gs jz 0x71	
Das	
jnc 0x79	
Popad	
fs outsd	
ja 0x16	


pop ebx - From using call, the address of the  next instruction is pushed to the stack, which for this will be a pointer to the 'pathname'. 

push dword 0x1b6 - Pushing 0x1b6 to stack which is 666 octal, which could be seen as the default value for the shellcode within msfvenom

pop ecx - Popping 0x1b6 to ecx for the 'mode' parameter

int 0x80 - Interupt to call the syscall

push byte +0x1 - Pushing 1 to the stack, to be popped into eax for the exit syscall

pop eax - The value 1 popped into eax for the exit syscall

int 0x80 - Exit is called and the program exited gracefully
```

Strace
------

Strace was utilised to observe the syscalls used and the parameters. The command used and screenshot of the output can be observed below:

	msfvenom -p linux/x86/chmod  -f elf > chmodelf
	
![strace](https://raw.githubusercontent.com/14Deep/14deep.github.io/master/_posts/Images/EX5/part1/strace.png)


GDB
------

To observe the shellcode executing within GDB, the elf payload created will be analysed. The command used to create this code is as follows:

	msfvenom -p linux/x86/chmod  -f elf > chmodelf

With the created payload, there are no symbols within the binary and so 'readelf' has to be used to find the entry point for the binary. This can be run as follows:

	Readelf --headers ./chmodelf

The output of this is as below:

![readelf](https://raw.githubusercontent.com/14Deep/14deep.github.io/master/_posts/Images/EX5/part1/readelf.png)


As can be seen, the entrypoint in this instance would be the address '0x8048054'. This can now be observed within GDB when setting a breakpoint at this address and running the program:

![instructions](https://raw.githubusercontent.com/14Deep/14deep.github.io/master/_posts/Images/EX5/part1/gdbinstructions.png)


Just before the first syscall for 'chmod', the registers were examined to determine if they were as would be expected from the earlier analysis:

![syscall](https://raw.githubusercontent.com/14Deep/14deep.github.io/master/_posts/Images/EX5/part1/chmodsyscall.png)

As expected, ebx contains the file to be manipulated:

![ebx](https://raw.githubusercontent.com/14Deep/14deep.github.io/master/_posts/Images/EX5/part1/ebx.png)

After the syscall is called, the program exits gracefully with 'exit'. 


This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert Certification:

http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert

Student ID: 1092
