**Paper:** [Improving The Reliability of Commodity Operating Systems](https://dl.acm.org/doi/pdf/10.1145/945445.945466)

# Intro.

While the OS kernel itself tends to be very reliable and well organized (because it is usually written by very experienced programmers), the *extensions* to that OS (e.g. *modules* in Linux and *Drivers* in Windows) are not particularly so.
Faults in these codes are one of the main reasons for Kernel crashes and failures. 

## Example: Segmentation Faults

We have all seen a SegFault, it tends to happen when you access an invalid memory address (for example, if you are in the User Space, you cannot access kernel memory addresses, and if you are the kernel, you can't access a memory address you have never initialized).

Usually SegFault causes:
- A process force kill, if it is a user space program. These days, the CPU itself will detect SegFauts:
	- It will then invoke the OS interrupt handler
	- The interrupt handler terminates the faulty process
- If it happens in the kernel though, we are *screwed*. The kernel will crash and will need to be restarted if the particular address space was not protected.

Why be so harsh? There is no way to tell whether or not only that single address was invalid (like a Null pointer exception) or if the *whole* address space has been corrupted (e.g. Memory Corruption bug). Assuming the worse would mean that proceeding with OS execution can cause irreversible damage to persistent data and that can get out of hand very quickly.

## Drivers

Kernel programmers are too busy making sure the kernel itself is bug-free. The rest of the kernel capability extensions (like say drivers for peripherals, keyboards, NICs, etc.) are written by 3rd party developers, which while could be very talented, they *didn't write the OS*, so their understanding of how it works could be wrong.

Beyond that, these drivers *assume* things about the hardware, so what if that assumption is wrong? The user can be source of error here, so how do we handle that?

It is important to note that at the time this paper was written (2005), more than 70 percent of the OS kernel faults were written So here is the challenge, **How to safely interact with possibly buggy code in the kernel space?**

### Naïve Solution

One way would be to give a driver its own address space, problem solved!
Well, not really. There are many problems with this solution:
- There are high communication overheads. No direct memory access should be allowed between them, instead, communicating processes in the kernel with the buggy code should use a shared memory space for communication. 
- Another way would be Pipes, or RPCs.

The things is, all of these at least have a context switch overhead and buffering delays. 

So we are forced to accept that these buggy code **will execute in the OS address space**. 
We could ask the OS devs to reprogram the OS with a type-safe programming language that is safe, but who would do that? There are millions of lines of code and thousands of drivers to rewrite!

## Nooks

The authors introduce **Nooks**, a *wrapper* around the OS kernel code that will enable the OS to interact with drivers safely. The main insight to how this works is by defining a limited, but well-defined interface between the OS and the drivers.

We have seen this before, using *polymorphic functions*, each driver will export an array of function pointers with well-defined names and actions for the OS to use, and the OS will be careful in executing them.

For example, to access a memory address in a disk, the driver should export a `read()` and `write()` function for the OS to use.

# Background

## RPCs

A Remote Procedure Call (RPC) is an abstract function that executes code in another address space. For example, the simplest RPC would be a wrapper around a socket `send` and `receive` that allows you to send a message to a server and ask it to execute something.

The abstraction that an RPC gives to a programmer is:
- A client sending a request should be equivalent to a **function call**
- A client receiving something from a server should be equivalent to a **function return value**
- A server receiving a request from a client should be equivalent to a **callback or function invocation**
- A server responding to a client request should be equivalent to **returning from callback to caller**

Thus, an RPC should provide an interface for these functions.

![[Pasted image 20240204190112.png]]

Usually we don't want to modify the client and server code too much, so we usually add separate codes to them via an imported module which we call a **stub**. These codes tend to be completely auto-generated and from the client/server perspective, they are just making a local function call and waiting for it to return.

It is important to stress that since this interface is very well defined, the stubs can be **auto generated**. They only require a description as input, usually the description of the communication protocol between the stubs and the type of messages.

## Extension Procedure Calls (XPC)

The solution Nooks uses, is a type of special RPC implementation called XPC. 

![[Pasted image 20240204210358.png]]

The key here is the ***wrapper*** code.
- It checks whether or not the function pointer passed by the kernel/XPC is actually valid.
- It ***copies*** the data from the OS address space and passes only that copy to the extension.

We must stress that this is a context switch between two **kernel level** processes, both running with the highest privilege. The context switching here though is similar to any other context switch. We change where the stack pointer is and switch the page table base by using the value in the CPU register.

## Wrappers

A wrapper first works by isolating kernel and extension address space. It does this by:
- Allocating separate address space for kernel and extension code, and creating separate page tables for them.
- It then protects them:
	- The kernel will have R/W access to its own space, as well as the extension space
	- The extension can only read the kernel space, it can do whatever it wants with its own address space (including corrupting it!)

Since the page tables are isolated, there is no way for any memory corruption in the extension space to leak into the kernel space.

>[!NOTE]
>While this works, it won't work if the driver deliberately tries to access the kernel space!
>Again, note that all of these extra wrapper codes are still being executed with the highest level of privilege, so it can *override* whatever page table it is currently using.

# Impact

While it is hard to say how much Nook impacted current OS implementation, it most definitely had very measurable effect on its own test cases. 
But it has its own limitations. While it is decent in protecting the kernel, it cannot do much in protecting the *hardware* itself. 

For example, if the bug happens in the middle of a disk operation such that by quitting and restarting, the file structure on the disk is left in an inconsistent state.
This is where Nooks abstraction creates trouble, it has no idea what the driver is actually doing, so it can't take measures to protect any thing other than the address space.

## Performance

![[Pasted image 20240204215238.png]]

Above you can see different benchmarks with and without Nooks.
It is fairly obvious that the overhead depends very much on how many XDC calls happen per second. For example:
- A low rate stream of XDC calls for something like sound leads to low overhead
- For network streams, sending has much more overhead compared to receiving. The reason is that sending data is just one call to send a portion of data, whereas receiving data can be done in batches.
- CPU intensive tasks tend to get hit harder
- Even if the CPU utilization is high, the overhead can be higher. The reason goes back to the page table switch. In CS 402, we saw that the CPU utilizes a TLB for caching page tables to reduce the overhead of using a page table in the first place. Each time change the table, we need to flush this buffer, and that takes time. Thus, Nooks heavily reduces the chance of page hit on the TLB, and that means that it is much more likely that we get a miss, and we will have to:
	- Get a new page table entry
	- Cache it in the TLB
	- Restarted the call (which by now should have received a page fault)
