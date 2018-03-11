---
layout: post
title: SLAE Exercise 3 - Egg Hunter
subtitle: Linux x86 Egg Hunter
tags: [SLAE]
---

**Overview**

The third task of the SLAE exam required the 'Egg Hunter' to be studied and written. The requirements for this were as follows:

- Create a working demo of the Egghunter
- Should be configurable for different payloads

**Requirements**

*What is an Egg Hunter?* 

An egg hunter is a technique that can be used to search the running programs memory range for an  further injected shellcode and redirect the execution flow to it. The egg hunter is ultimately shellcode itself, used as a means to execute further shellcode. 

*How does this help?*

The main advantage of this is the size of the egg hunter. This allows vulnerabilities whereby there is not enough space for full shellcode to be placed to still be exploited. 

*How is this done?*

An 8 byte key is placed at the start of the larger injected  shellcode that would not fit into the original buffer, this is to be as unique as possible so that the same instructions would not be accidently run into. 

*How can this be created?*

To create this, our egg hunter is required to search the programs memory for our key at the start of the main shellcode. This can be done multiple ways using different syscalls. 



To do this exercise, the Skape 'Safely Searching Process Virtual Address Space' egg hunter paper was a brilliant reference for different egg hunter techniques and implementations. This can be found [here](http://www.hick.org/code/skape/papers/egghunt-shellcode.pdf).

This paper provides multiple different methods of implementing an egg hunter in linux. For this example, the first presented method will be utilised which takes advantage of the 'access(2)' syscall. 


**Access(2) syscall**
The access(2) syscall as described in the man pages is usually used to check whether the calling process can access the file 'pathname' - checking to see if the process has adequate rights to a given file.  As can be seen in the man pages, the syscall requires the parameters listed below:

```
	int access(const char *pathname, int mode);
```

As can be imagined from what the syscall actually does, the 'pathname' pointer is the parameter that will hold the address required to check, in this scenario this will be the egg hunter key. The 'mode' integer specifies the check to be performed against the 'pathname' value, this is not required for this shellcode. 

As before, the syscall can be located within the following file (this may vary depending on the system being used):

	/usr/include/i386-linux-gnu/asm/unistd_32.h

This shows that the access syscall is number 33 - '#define __NR_access 33'. 


**Safely Searching Process Virtual Address Space PoC**

The paper previously mentioned contains a proof of concept for each of the suggested egg hunter methods. Below is this code:

  *int access(const char *pathname, int mode);*

```
mov ebx, 0x50905090
xor ecx,ecx
mul ecx
or dx, 0xfff
inc edx
pusha
lea ebx, [edx+0x4]
mov al, 0x21
int 0x80
cmp al, 0xf2
popa
jz 0x9
cmp [edx], ebx
jnz 0xe
cmp [edx+0x4], ebx
jnz 0xe
jmp edx
```

**Understanding the code**

It is required to fully understand the code and what is happening to be able to write a version of the egg hunter. Some points were clearer then others, with some code having to be written and debugged to understand what was happening. 

*Initalization*

mov ebx, 0x50905090 - This instruction moves the chosen egg hunter key into the ebx register. This is the value that the egg hunter is attempting to locate within memory and will indicate the start of the main shellcode. 

xor ecx,ecx - This is initializing the ecx register by clearing it to 0. 

mul ecx - Again, this is initializing a register however is done in a slightly different way. By default if one parameter is provided to mul it will multiply the supplied value to ax. In this case eax was cleared to 0 in the instruction before therefore when multiplying ecx by 0, the value will always be 0. 


*Page alignment*

This is used to iterate through the programs virtual address space, incrementing by 0x1000 whilst comparing the value with the egg hunter key. A page is a fixed length block of memory which is the smallest unit of data for memory management, therefore iterating through these blocks should at some point identify the key. 

or dx, 0xfff- The value of dx (which on the first iteration will be 0) is used with the 'OR' operation with 0xfff. In this program, edx will be used as a pointer to a memory location. A page is a fixed length block of memory which is the smallest unit of data for memory management, therefore iterating through these blocks should at some point identify the key. 

Inc edx - The value oxfff has essentially been inserted into dx within the instruction before by using the 'or' operator. However 0x1000 is required for the page alignment operation. The edx register is therefore incremented by 1 to make the value of edx 0x1000. A subset of the shellcode was adapted and written into a loop to show this easily when debugging with GDB. The code written was:

![Page Alignment]()


