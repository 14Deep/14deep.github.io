---
layout: post
title: SLAE Exercise 5 (part 2) - Read File
subtitle: Shellcode Analysis 2 - Read File
tags: [SLAE]
---

linux/x86/read_file
======

The second chosen shellcode to be used from msfvenom is the 'read_file' payload. This, as expected, reads a specified file and displays it on the screen. The parameters to be used for this payload can be seen with the following command:

	msfvenom -p linux/x86/read_file --payload-options
	
The command that is used to create a payload that will read the '/etc/passwd' file is as below:

	msfvenom -p linux/x86/read_file PATH=/etc/passwd -f raw > readfileraw

Once this has been generated it can be disassembled using ndisasm to analyse the instructions used with the following command:

	ndisasm -b 32 ./readfileraw

![Ndisasm](https://raw.githubusercontent.com/14Deep/14deep.github.io/master/_posts/Images/EX5/part2/Ndisasm.png)

Stepping through the instructions
------

Each of the instructions seen here will be stepped through, describing what is happening at each step. 



**open**  

int open(const char *pathname, int flags);

The open syscall opens the file that is specified by 'pathname', if the file exists the return value of the syscall is a file descriptor used in subsequent syscalls to refer to this opened file. 


jmp short 0x38 - jumps to location 0x38, which in this instance calls 0x2 (the next instruction). This is used as a 'jmp call pop' to push the next instruction after call onto the stack, to be used as the pathname pointer.The instructions for this section are as follows:

	mov eax, 0x5 - move 5 to eax, used for the open(2) syscall

	pop ebx - pop value in stack to ebx which should contain the pathname. 

	xor ecx, ecx - clear the ecx register 

	int 0x80 - interrupt to call syscall



**read**  

ssize_t read(int fd, void *buf, size_t count);

The 'read' syscall will read bytes from the file descriptor retruned from 'open' into the buffer that is selected as a parameter within the ecx register for this syscall. The return value is the number of bytes that have been read which can be used for the count parameter within the next syscall. The instructions for this section are as follows:


	mov ebx, eax - move the return value into ebx (the fd)

	mov eax, 0x3 - move 3 into eax for the read(2) syscall

	mov edi, esp - move stack pointer to edi

	mov ecx, edi - then move the value from the stack pointer into ecx for the syscall

	mov edx, 0x1000 - move 0x1000 into edx

	int 0x80 - interrupt to call syscall


 
**write**  

ssize_t write(int fd, const void *buf, size_t count);

The 'write' syscall writes up to the 'count' value of bytes determined within edx which in this case is will be the return balue from the 'read' syscall. This will be written to the file descriptor selected within the first parameter, which for this will be 1 for STDOUT to display the contents of the chosen file. The instructions for this section are as follows:

	mov edx, eax - move return value to edx (number of bytes read is returned to eax)

	mov eax, 0x4 - move 4 to eax for write(2) syscall

	mov ebx, 0x1 - move 1 to ebx

	int 0x80 - interrupt to call syscall



**exit**
The instructions for this section are as follows:

	mov eax, 0x1 - move 1 to eax (exit syscall to exit gracefully)

	mov ebx, 0x0 - move 0 to ebx

	int 0x80 - interrupt to call syscall




Strace
------

Strace was utilised to observe the syscalls used and the parameters. The command used and screenshot of the output can be observed below:

	msfvenom -p linux/x86/read_file PATH=/etc/passwd -f elf > readfileelf

![Strace](https://raw.githubusercontent.com/14Deep/14deep.github.io/master/_posts/Images/EX5/part2/strace.png)


GDB
------

To observe the shellcode executing within GDB, the elf payload created  will be analysed. The command used to create this code is as follows:

	msfvenom -p linux/x86/read_file PATH=/etc/passwd --f elf  > readfileelf

With the created payload, there are no symbols within the binary and so 'readelf' has to be used to find the entry point for the binary. This can be run as follows:

	Readelf --headers ./readfileelf

The output of this is as below:

![readelf](https://raw.githubusercontent.com/14Deep/14deep.github.io/master/_posts/Images/EX5/part2/readelf.png)

As can be seen, the entrypoint in this instance would be the address '0x8048054'. This can now be observed within GDB when setting a breakpoint at this address and running the program:

![entrypoint](https://raw.githubusercontent.com/14Deep/14deep.github.io/master/_posts/Images/EX5/part2/entrypoint.png)

The intial 'jmp call pop' can be observed initially, afterwards the parameters of the first syscall are then moved into the appropriate registers:

![entrypoint](https://raw.githubusercontent.com/14Deep/14deep.github.io/master/_posts/Images/EX5/part2/jmpcallpop.png)

This allows the location of the 'pathname' parameter to be pushed to the stack which will subsequently be popped into ebx in the next few instructions. 


**Open**

Just before the first interrupt, the registers are as follows:

![open](https://raw.githubusercontent.com/14Deep/14deep.github.io/master/_posts/Images/EX5/part2/openregisters.png)

As can be seen in the 'open' man pages, the parameters required for the open syscall are:

	int open(const char *pathname, int flags);

This can be compared with the values in the registers above, where we can see that:

- eax is 5 for the open syscall
- ebx contains the pathname, the address of which was pushed to the stack as part of the earlier 'call'.
- ecx contains 0 as no flags are required


**Read**

Just before the second interrupt, the registers are as follows:

![read](https://raw.githubusercontent.com/14Deep/14deep.github.io/master/_posts/Images/EX5/part2/readregisters.png)

Again, as can be seen in the 'read' man pages, the parameters required for the read syscall are:

	ssize_t read(int fd, void *buf, size_t count);

This can be compared with the values in the registers above, where we can see that:

- eax contains 3 for the 'read' syscall
- ebx contains 3 for the fd which was returned from the previous call
- ecx contains the location to read bytes from 
- edx contains 4096, the amount of bytes to read up to (page)
	
**Write**

For the write interrupt, the registers are as follows:

![Write](https://raw.githubusercontent.com/14Deep/14deep.github.io/master/_posts/Images/EX5/part2/writeregisters.png)

Again, as can be seen in the 'write' man pages, the parameters required for the write syscall are:

	ssize_t write(int fd, const void *buf, size_t count);
	
This can be compared with the values in the registers above, where we can see that:

- eax contains 4 for the write syscall
- ebx contains 1 for the stdout file descriptor 
- ecx contains a pointer to the buffer (0xbffff070)


**Exit**
The rest of the shellcode gracefully exits the program which will not require walking through. 



This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert Certification:

http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert

Student ID: 1092





