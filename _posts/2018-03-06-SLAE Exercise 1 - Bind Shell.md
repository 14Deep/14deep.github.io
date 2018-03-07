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



