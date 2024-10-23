**Note: Skipped File System (Part 1)**

# Virtual Memory

Main Challenges of Virtual Memory Implementation:
- What to do when the user space requests more memory than actually available?
- How would we implement protection? i.e.:
	- How can we protect a user process's memory space from another user process
	- How can we protect kernel's *own* address space from users?

We already know the basic tool that allows us to tackle all the problems above, that being the Address Space Abstraction, which:
- Protects processes, as it's impossible for different processes to cross into each other's address space.
- Virtual Memory Map allows us to keep using virtual memory maps, even if the physical memory has been exhausted.

## Virtual Address

The abstraction is that:
- All applications (even kernel processes themselves) use virtual addresses in a virtual memory to access physical memory. 
- All things directly using the BUS connected to the CPU and devices, as such:
	- Devices use *only* physical addresses when communicating with the CPU
	- The CPU handles both physical address and virtual addresses
	- When interacting with the CPU and memory, **all** software may only use virtual addresses

As such, the OS itself:
- Cannot use physical addresses, only pass them around
- The OS may only *manage* physical address, but not use them directly (like the buddy system for example)

The situation deep in the CPU is quite complicated and we won't go too deep into it, but just let us say that almost *everything* we saw in the CPU used purely virtual addresses (like for example, the EIP, ESP and EAX all use virtual addresses).

The process of turning Virtual Address to Physical Address is called **Address Translation** and is handled partly by a piece of hardware in the core of the CPU called the **Memory Management Unit** or ***MMU***. 

![[Pasted image 20230409214116.png]]

The MMU accepts a V.A. from a particular process and converts it to a P.A. in the memory. As such, a V.A. alone is not enough to specify a physical memory location, it depends on *which* process has asked for it!

>[!NOTE]
>As the MMU is using P.As, it is obvious that it ***SHOULD*** be connected to the bus. Also, the MMU has it's own registers in the MMU, but we haven't seen them yet, since we have only discussed registers used by programs, not hardware.
>
>One important MMU register, is the aptly named **MMU Register**, which will contain the base physical address of the address space for the current process, and as such, the P.A. associated with an input V.A. can be calculated by just adding the value of this register to the V.A.

### Memory Fence

One really old implementation of the MMU is the memory fence:

![[Pasted image 20230409214631.png]]

Here, the OS and user address space will be separated by pre-defined border, where the MMU will generate a trap if a user space code attempts to reach the OS part. However, it provides no protection for user processes themselves. As such, this things does not even use V.A. really, as P.A. is just passed around as is.

**Note:** In the picture above and any that follows, the "Inner Core" is the part of the CPU that is using V.As.

