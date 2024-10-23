# Context Switching (cont.)

## System Call Context Switching (cont.)

Last lecture, we left off after discussing how System Calls are handled, and we finished by setting where and with what function context switching would be required. Now, we want to discuss the details of that process, and in order to do that, we need to get familiar with the `trap` machine instruction.

### `trap` Machine Instruction

The machine instruction `trap` does the following:

1. Traps into kernel by **Disabling Interrupts**, by setting the flag in the CPU and switching to **Privileged Mode** by also setting the control bit in the CPU.
2. The kernel allocates "temporary locations" that will save the context of the entire CPU (i.e. all registers at the least). The location is CPU dependent, and the data structure will be available to all kernel threads.
   Some CPUs call this temporary location the **Interrupt Stack**, which is a data structure shared by ALL interrupt service routines, and thus needs to be regularly cleared and freed.
   To hide these CPU dependent procedures from the kernel (which needs to be CPU independent as much as possible), the hardware will have a piece of code as firmware called **Hardware Abstraction Layer (HAL)**, which is outside the scope of this class.
   Just know that all CPU dependent code is consolidated here, and we don't need to worry about it.
3. The HAL will now set the SP in the CPU to the top of the kernel stack corresponding to the user thread that tried to execute the `trap` instruction. (remember, all user processes have **both** user and kernel stacks allocated for them by the OS).
   As to where the pointer to this stack is located, it is located in the Process Control Block of the user process by default.
4. HAL sets the EIP in the CPU to point to `interrupt_handler`, which is C code in the OS specifically designed to handle interrupts. Now, we copy ESP and EIP stored in the "temporary location" in step 2, and we push them into the kernel stack.
   Thus, **the bottom of the kernel stack of a user thread will save the user thread context**.
   Once this is done, interrupt is re-enabled.
   Now, we will execute code specifically meant for that particular trap (i.e. one of 256 actual Syscalls).
5. Once we return from the interrupt handler, we perform a special `return` instruction (`iret` in Intel x86), which in terms of function, is the above steps, but backwards (so yes, we need to disable interrupts again!). 
   ***This code is executed by HAL***.
   
>[!NOTE]
>The above instruction is the ONLY way to go back from kernel space to user space.
>Trap is also the ONLY way you can go from user space to kernel space.

>[!FAQ] What About Hardware Interrupts?
>Hardware interrupts are handled similarly. We'll see it later.

>[!Note] Interrupt Context
>In OS jargon, everything done by HAL is referred to as the Interrupt Context.
>
>When executing `trap`, you start from user context, switch to interrupt context, execute code in kernel context, return to interrupt context to get ready to leave kernel space, and then return to the user space and resume execution.
>
>That said, there is still weird cases between these context switches that you are technically in **both** or in **neither**! Context Switching really isn't clean business.


## Interrupt Context Switching

When we get a hardware interrupt, we can be either in kernel code, user code, or even the middle of an interrupt service routine.

Before we proceed, make sure that you understand that we are ***NOT*** talking about signals.
Signals are generated **by the kernel** and then **delivered by the user code**, and thus are not the same as software interrupts.
Neither are they hardware interrupts, which is our main focus here (for obvious reasons!), which are delivered into the kernel after passing the HAL.

Some hardware interrupts are dire and important, thus we should be able to completely stop any running process (regardless of context) and just move in and handle the hardware interrupt that HAL delivered to us.

As we said, hardware interrupts are handled quite similar to software interrupts, but with some key differences:
- In step 3, which kernel stack should we use??? We should discuss that.
- In step 4, after re-enabling interrupts (after masking/blocking some interrupts), we stay in the interrupt context (i.e. in the HAL), we don't return to thread context and execute kernel code.

The stack used for interrupt service routine is a design choice. You can:

1. Allocate stack from memory each time an interrupt happens.
   This is too slow.
2. Have a stack shared by ALL interrupts
   Not often done, it's not really clean
3. *Borrow* a kernel stack from the thread that is being interrupted!
   **Most Common!**

Why option 3 is nice?
- If we interrupt in the user space, we can be sure that kernel stack assigned to the process is **empty**! 
  The stack is nice and clean, so it makes sense to do this.

#### Interrupt In User Space

To find the stack, we can use the global variable `currentThread` which points to the TCB of the thread currently running. Thus, when an interrupt happens in user space, we borrow the kernel stack of `currentThread`, push the user thread context in the bottom and start using the rest of the stack to execute C code in HAL.

#### Interrupt In Kernel Space

**What if we are interrupted in kernel space?**
Most of the time, hardware interrupts are much more important than kernel thread code execution. Thus, we do this:
- We save the kernel thread context, **ON TOP** of the current kernel stack.
- We start using the rest of the kernel stack to execute C code.

So schematically, it would look like this:

![[Pasted image 20230304031021.png]]

#### Interrupt In Interrupt Service Routine

Depending on whether or not we consider the new interrupt to be more important than current interrupt (which is done by checking interrupt priorities), we do exact same as the previous case if we decide to switch to the new interrupt.

![[Pasted image 20230304031326.png]]

There is one important thing though …
Once we are in the new interrupt handler, we **NEVER** return to any other context, until we are finished.

Why?
- If we switch to a lower interrupt context, that context can make function calls and corrupt the stack of the previous interrupt (remember, the new one is right on top of the previous interrupt)
- If we switch to the kernel context, the same thing can happen for both the previous interrupt and the original lower priority interrupt (since they are both on top of the kernel stack frames)
- We also can't even switch to any other user space context!!
  Why? Well, if that other context decides to switch back to us and execute a Syscall, it will wipe out the current kernel stack!

Thus:

>[!IMPORTANT]
>Once we have interrupt handlers running, all normal threads, no matter how urgent, need to wait for *ALL* interrupt handlers to finish.
>
>Thus, it is paramount that interrupt handlers execute as little code as possible.

You can remedy this by choosing approach (1) for choosing stacks for interrupts (i.e. allocate fresh stacks), but that's just too slow!

>[!FAQ] What About I/O?
>I/O interrupts are notoriously slow, how would we remedy that if we need to be as fast as possible in interrupt contexts?
>
>Well, the key is to defer the completion of that interrupt to some other time. We are yet to see how this is done, but just know that if the I/O decides that it's going to take time, any thread waiting for it should just go to sleep (or the user should code it some way that it will do other useful things) and pass the CPU to other things.
>Once the I/O completion interrupt happens, it's like a `pthread_cond_signal`, *someone* was waiting for this at some point, thus we wake that person up by moving them out of the queue of sleeping threads.

>[!NOTE] Hardware Interrupt Masking
>The CPU should disable all interrupts when handling critical interrupts of high priority.
>
>There is two common approaches for enabling/disabling interrupts:
>
>- **Hardware Register Implementation:** Use a bit vector for masking particular interrupts (similar to how we did it for signals).
>  This is difficult to use and program on, thus it's not used much anymore (with the exception of Intel!)
>- **Hierarchical Interrupt Levels:** (More Common!) The processor masks interrupts by setting an **Interrupt Priority Level (IPL)** (just a number) in some particular register.
>  When we are servicing, any interrupt with priority lower than or equal to the IPL will be blocked.
>  So, if you would set the IPL to 7, interrupts from 8 and above can still be seen.
>  
>  For doing particular interrupts, we set the IPL to that of the current interrupt, do our job to handle it and upon return, we reset the IPL back to it's original value.
>  All that a programmer needs to do, is to assign priorities to each particular interrupt.
>  
>  It's funny to note that even if someone insists on using (1), people will abstract it away by implementing code in HAL that makes it look like (2).

# I/O Architecture

The **Memory Mapped I/O** architecture is the one we use in this class, where the CPU pretends that all devices are essentially memories on a bus.

![[Pasted image 20230305015158.png]]


There are two broad category of devices here:

- **Programmed I/O (PIO):** These devices start a bus cycle that reads/writes directly to the CPU registers.
  For example, to see what keyboard button was pressed, the CPU will perform a read operation directly. These tend to be slow.
- **Direct Memory Access (DMA):** The controller device is the one that actually performs the I/O operation. The CPU only specifies to **where** or **when** and gives a **GO** signal for it to start.
  These controllers can be fast, since unlike PIO devices that transfer byte per byte, these can transfer huge amounts of data in one go.
  The best example of this is RAM.

Registers used for PIO devices are just memory locations with special semantics and names.
The following schematic gives you a general idea about this.

![[Pasted image 20230305015844.png]]

>[!IMPORTANT]
>To actually work with PIO, since they are slow, we ***NEED*** to interrupt the CPU.

The same is true for DMA devices:

![[Pasted image 20230305020002.png]]

As you can see, these devices have bigger memory and device addresses and thus, can transfer larger amounts of data.

## Drivers

As you can see above, actually using these devices is hardware dependent and really, *really* tedious. The kernel developer really wouldn't like writing I/O code. We also want for our kernel to be device independent, so how can we handle this?

Well, we ask the **device manufacturer** to implement corresponding functions needed to work with the device, and provide an interface for our OS. 

The hardware engineers package these codes into **Device Drivers** and hand them out to kernel engineers so that they can go about doing their business. The device deriver engineer will also take great care about how they implement functions so that the OS can treat I/O ***independent of device***.

To do this, kernel engineers specify a well-formed array of pointers to functions for the hardware engineers which provide the API that we need (these can be simple read/write operations), the hardware engineers will then write code specifically for these APIs and thus the kernel engineer can  just use this API anywhere, regardless of what device is being used.

![[Pasted image 20230305020404.png]]

For example, as a C++ interface, what the kernel engineer would use to work with disk devices is something like this:
```c++
class disk {
	public:
		virtual status_t read(request_t) = 0;
		virtual status_t write(request_t) = 0;
		virtual status_t interrupt() = 0;
};
```
These functions as you can see, have no implementation right now! 
Once the drivers are integrated into the kernel, appropriate byte code will be generated by the driver that the kernel can then link to these functions so that we can use disks as we need, but **not a single line of actual kernel code needs to be changed!**

As for the hardware engineer, they first implement their desired code in whatever way they want, and pack them in an array of function pointers, which can look something like this for a hypothetical engineer:
```c++
(void *) weyland_yutani_disk_ops[] = {
	read_handler_t weyland_yutani_read;
	write_handler_t weyland_yutani_write;
	intr_handler_t weyland_yutani_intr;
	...
}
```
And then, in the last stage, they **bind** these pointers to appropriate kernel functions like the `disk` class that we saw above:
```c++
disk d;
d.read = wayland_yutani_disk_ops[0];
d.write = wayland_yutani_disk_ops[1];
...
```
This finally solves this problem.

>[!NOTE] Asynchronous Operations
>Most I/O operations are implemented asynchronously, and this goes for internal driver implementations as well.
>
>More realistically, the interface for drivers looks much more like the file system Syscalls that we've used, in the sense that they return a **handle**, identifying that the operation has started.
>Threads however, can use the equivalent of `wait` in the operations provided to wait for these operations synchronously.
>
>Generally, asynchronous interface is just more natural for hardware, as they run in parallel with the CPU.

# Dynamic Storage Allocation

In the user space, we need a ***memory allocator*** for populating the heap or maintaining persistent variables.

It's important to stress that while "memory allocator" really sounds like a process, it's nothing but just a set of two functions, `free` and `malloc`, there is no thread or process involved!

We use these functions to maintain memory blocks (which are nothing but a contiguous array of memory addresses) under the following **contract**:

- The application (i.e. the entity calling the memory allocator) never touches anything owned by the memory allocator.
- The application *ONLY* touches it's own memory blocks, which ***MUST*** be allocated strictly by the memory allocator.
- The memory allocator *guarantees* that it will not touch any block of allocated memory, unless called upon specifically to do just that.

Any violation of these rules will create a **Memory Corruption Bug**; after this happens, all guarantees made by the memory allocator are **no longer true**, which means that a call to either `free` or `malloc` can either crash another process, or just outright fail!

## Memory Allocator Implementation

There are two abstract implementations of the memory allocator, ***best-fit*** and ***first-fit***.

The allocator maintains a global linked list, that points to empty spaces of the memory. When `malloc` is called to allocate `n` bytes:

- For **best-fit**, we search the whole list for the block that is as close as possible to `n`, or choose the first one if multiple ones exist.
- For **first-fit**, we just use the first block that is at least `n` bytes.

Schematically, it looks like this:

![[Pasted image 20230305031331.png]]

As you can see, **best-fit** just failed in the end! Turns out, there is a more deeper pattern here.

>[!IMPORTANT]
>Assuming programs use a pretty random allocation pattern (which is quite reasonable), it's easy to see that for the same string of allocation:
>
>- Best-Fit and First-Fit will end up with the same average amount of free memory.
>- Best-Fit will have a lower variation in free block sizes, compared to First-Fit.
>
>Ideally, we want large average block sizes so that we can store larger things, but we also want *larger variation*, because programs may require wildly different memory sizes at any point!
>Thus, we actually prefer First-Fit for `malloc`.

>[!NOTE] Fragmentation
>A problem that happens when using Best-Fit is that it leaves a large number of small blocks. To formalize what we mean, we define the following:
>
>- **Internal Fragmentation:** Unusable memory contained within an allocated region.
>- **External Fragmentation:** Unusable memory contained in small blocks over a large area.
>
>As you can see, Best-Fit suffers quite a lot from external fragmentation, however there is barely any internal fragmentation. Turns out there is a way to reverse this with another allocation method, the **buddy system**.


 