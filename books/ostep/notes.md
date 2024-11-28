# Operating Systems: Three Easy Pieces

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

### Why is File Sytem not a library? (ie Process Call Vs System Call)

Code ran on behalf of a OS is special as it has control of devices. To ensure protection (isolation), we need to make a distinction between application code and OS code. Hence, system calls were invented. 

Main difference between a system call and a process call? 
A sys call transfers control over to the OS and raises the hardware privilege level. 

How does a sys call work? 
1. sys call is initiated through a special hardware instruction called _trap_. 
2. hardware transfers control to a per specified _trap handler_ (that OS setup previously) and raises privilege to kernel mode. 
3. services request. 
4. when finished 3, control is passed back to the user using _return-from-trap_ instruction. 

Kernel Mode: OS has full access to the hardware of the system and thus can do things like initiate an I/O request or make more memory available to a program

## Process  

CPU creates an illusion of infinite number of CPUs by using time sharing. 

Time sharing - a resource is used by one entity for a time and then by another entity, and so forth. performance is affected for each entity. eg - CPU, network link. 

Space sharing - resource is divided among entities. eg - disk 

### How is virtualisation implemented by the CPU 
A low level machinery and high level intelligence is needed. 

Mechanisms - low level methods or protocols used to implement a feature. eg - to implement context switch, time sharing mechanism is used. answers a _how_ question. 

Policy - the high level intelligence. Algorithms needed to make decisions with the OS. eg - scheduling policy. answers a _which_ question.  

The two are spearated to enhance modularity in the system. This way policies can be changed without changing the underlying mechanisms. 

### What constitues a process? 

A process is simply a running program. Taking an inventory of all systems it access or affects during its execution, ie its machine state, gives a complete picture of the process.

Machine State - What a program can read or update when it is running.

A process's machine state constitues of: 
1. Memory - ie the address space. Program instructions + data the process reads and writes.
2. Registers - all registers that process instructions read or update. Including special registers: 
    a. Instruction Pointer (IP) or Program Counter (PC)
    b. Stack Pointer 
    c. Frame Pointer 
3. Persistent Storage Devices - eg the files currently open by the process. 

### How is a Process created by the OS? 

1. Lazy load the process code (and any static data like intitialised variables) from disk to the address space of the process. 
2. Allocated memory for program's run-time stack (for local variables, return addresses, func parameters) and intialise the stack with `main()` func arguments. 
3. May allocate some memory for program's heap. (small at first and depending on memory utilisation of program, OS may allocate more memory) 
4. Work related to I/O. 
5. Jumps to `main()`, the entry point of the process and tranfers control of the CPU to the newly initialised process. 

Lazy loading - Loading pieces of code or data only as they are needed during program execution.

### Different states possible 

1. Ready - ready to start execution but for some reason OS hasn't scheduled this yet. 
2. Running - process is exectuing instructions. 
3. Blocked - process is not ready to run due to some other operation. eg - waiting for I/O 
4. Zombie - process is in its final statewhere it has exited but not yet been cleaned up. 

Process List - List of all processes in the system. 
Process Control Block (PCB) - DS containing information about a process. 

### Datastructure for a process in Linux 

- different process states possible: https://github.com/torvalds/linux/blob/7eef7e306d3c40a0c5b9ff6adc9b273cc894dbd5/include/linux/sched.h#L87
- structure for a process: https://github.com/torvalds/linux/blob/7eef7e306d3c40a0c5b9ff6adc9b273cc894dbd5/include/linux/sched.h#L778
## Things to search: 
* Von Neumann model of computing 
* Address space randomisation 
* hardware instructions: trap, trap handler, 
* kernel mode 