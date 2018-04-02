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

![Page Alignment](https://raw.githubusercontent.com/14Deep/14deep.github.io/master/_posts/Images/EX3/incedx.png)

When debugged, the following shows the output of the 'or' instruction, followed by the 'inc' instruction:

![Page Alignment](https://raw.githubusercontent.com/14Deep/14deep.github.io/master/_posts/Images/EX3/incedx2.png)

Pusha - This instruction pushes all general purpose registers to the stack. This is used to maintain the values after the syscall is used. 

*Populating syscall registers*

lea ebx, [edx+0x4] - This instruction is used to load the address of edx+0x4 into ebx, which is the parameter of the syscall which is used for the 'pathname'. As discussed before when looking at the 'or' instruction, the value within edx is a pointer to a memory location that increases upon every iteration. The value 4 is added to the pointer as it will allow 8 bytes of memory to be validated with each call. 

mov al, 0x21 - This instruction is simply putting the value of the access(2) syscall into eax. 

int 0x80 - The syscall is now called with an interrupt. 


*Determining if the address was the required location*

cmp al, 0xf2 - The return value of the syscall will be places within eax. This instruction compares that value with 0xf2, which is the low byte of the EFAULT return value. As stated in the paper by Skape, when a system call encounters an invalid memory address, most will return the EFAULT 6 error code to indicate that a pointer provided to the system call was not valid. Therefore, if the values match, it is known that there was an error and the location did contain the egg hunter key. 

To prove that the return value for a failed attempt would contain 0xf2 within al the PoC code that was provided by Skape was written to work, compiled and debugged. Upon the syscall running and the search failing the following was observed:

![retval](https://raw.githubusercontent.com/14Deep/14deep.github.io/master/_posts/Images/EX3/retval.png)

popa - This is the opposite instruction to the 'pusha' instruction earlier, this pops all the general purposes registers that were saved in the stack earlier back to the registers. The flag from the instruction before will still be set, the important thing here is that the value of ebx would have changed back to the required egg hunter key to allow confirmation. 

jz 0x9 - Based on the result of the 'cmp' instruction seen earlier, a jump will take place if the zero flag is set. If the flag is set, which would imply that the comparison was a success and the location was not correctly identified then the flow would jump back to the 'or' operation described earlier.  


*Comparing the value at the location*

cmp [edx], ebx - It is now assumed that as no EFAULT was received that the location that has been checked is the location containing the egg hunter key. The value of ebx which was originally the egg hunter key that was to be searched for is compared with the contents of the pointer in edx, the current location that was checked using the access(2) syscall. 

jnz 0xe - If the zero flag is set, meaning that the values were not the same, then the flow would jump back to the 'or' operation described earlier to check the next address. 

cmp [edx+0x4], ebx - Another comparison is carried out, however this time it is with the value in edx+0x4. This is because the egg hunter key supplied would fill both locations with the repeating pattern. 

jnz 0xe - Once again, if the zero flag is set and they do not match the flow would jump back to the 'or' operation described earlier to check the next address. 

*Continuing the execution*

jmp edx - At this point the egg hunter has successfully located the rest of the shellcode. All that is left to do is to alter the flow of execution to the main shellcode. This is simply done by unconditionally jumping to edx, which is the location which has been identified as holding the egg hunter key. The main shellcode will then start to execute. 


**Writing an egg hunter**

Based on the Proof of Concept that was set out in the paper, an egg hunter shellcode was written. 

```

; Filename: egghunter.nasm
; Author:   Jake
; Website:  http://github.com/14deep
; Purpose:  Egg Hunter shellcode using the access(2) syscall


global _start			

section .text
_start:


	; Clearing all the registers that will be used and moving the egghunter key to ebx. 
	; Note this value is 1 larger than the required key and then decremented afterwards.
	; This is to stop the egghunter thinking thinking that this may be the key to find.
	
xor ecx, ecx			; value to be empty for syscall
xor edx, edx
mul ecx				; eax nulled
mov ebx, 0x50905091     	; key to find + 1
dec ebx 			; dec edx to = the key


addr_page:
or dx, 0xfff 			; edx is used as a pointer used to iterate through blocks of memory
				; See write up for a breakdown

incr:
inc edx				; increment edx to push it from 0xfff to 0x1000 - it will add 0x1000 each call
pusha        			; Push all general purpose registers to the stackto maintain state after syscall

lea ebx, [edx+0x4]		; loading the address of edx+0x4 to ebx for the addr parameter
mov al, 0x21 			; access(2) syscall value
int 0x80     			; Interupt for syscall


cmp al, 0xf2 			; Compare return value with 0xf2 - lowest byte of efault value 
popa         			; pop registers back from before syscall
jz short addr_page 		; If the zero flag is set, jump to addr_inc


cmp [edx], ebx 			; Compare the contents of ebx with edx, the key
jnz short incr	 		; If the values do not match a jump is taken to addr_inc


cmp [edx+0x4], ebx 		; Compare the contents of ebx+0x4 to edx, the key
jnz short incr	 		; If the values do not match a jump is taken to addr_inc


jmp edx      			; unconditional jump to edx, the confirmed location to continue execution

```


This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert Certification:

http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert

Student ID: 1092
