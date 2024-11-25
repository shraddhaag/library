## Operating Systems: Three Easy Pieces

The latest PDF version of the book is available (here)[https://pages.cs.wisc.edu/~remzi/OSTEP/].

Operating System: body of software making it easy to run programs, allowing programs to share memory, emabling programs to interact with devices, etc. Also referred to as: _virtual machine_ (as it virtualises resources), _standard library_ (as it exposes syscalls), _resource manager_ (as it manages shared resources across different applications), and earlier as _supervisor_ or _master control plane_. 

Virtualisation: OS takes a physical resource and transforms it into a more general, powerful and **easy to use** virtual form of itself.  

System Calls: To access virtual resources, OS exposes 100s of APIs, ie system calls, to applications. 

Virtualising the CPU: To seemingly run many programs at once, OS turns a single CPU (or a small set) into seemingly infinite number of CPUs. 

A program keeps all its data structures, program instructions in memory. 

Virtualising the Memory: Each process has its own private virtual address space which OS maps onto the physical memory. This is how memory references of a process does not interfere with that of another process. 

The problem of concurrency first arose in the OS as it needs to juggle doing many things at the same time. 

File system: software in OS that is responsible for managing the disk. 

OS does not create a private, virtualized disk for each application. 

### Design Goals of OS 

This may vary on the specific OS, but in general: 
* Abstractions for easy of use. 
* Highly performant ie minimize load on the OS
* Isolation: b/w application and also b/w applications and the OS. 
* Reliable 

## Why is File Sytem not a library? (ie Process Call Vs System Call)

Code ran on behalf of a OS is special as it has control of devices. To ensure protection (isolation), we need to make a distinction between application code and OS code. Hence, system calls were invented. 

Main difference between a system call and a process call? 
A sys call transfers control over to the OS and raises the hardware privilege level. 

How does a sys call work? 
1. sys call is initiated through a special hardware instruction called _trap_. 
2. hardware transfers control to a per specified _trap handler_ (that OS setup previously) and raises privilege to kernel mode. 
3. services request. 
4. when finished 3, control is passed back to the user using _return-from-trap_ instruction. 

Kernel Mode: OS has full access to the hardware of the system and thus can do things like initiate an I/O request or make more memory available to a program

## Things to search: 
* Von Neumann model of computing 
* Address space randomisation 
* hardware instructions: trap, trap handler, 
* kernel mode 