
# Handling Interrupts

Interrupts have associated functions that handle them, called interrupts handlers.
All interrupt handlers are held in a vector of function pointers, which are accessible by a function called an **interrupt service routine**. 

Interrupts are handled by deviating threads, potentially even kernel threads! So they have the same mechanism of handling like signals:

![[Pasted image 20240401232831.png|200]]

The main difference though, is that when the handler of interrupt with level $L$ is invoked, **all interrupts with levels** $L$ **and lower are disabled**. This means that only higher level interrupt handlers can interrupt another interrupt handler.

Interrupt handlers for this reason, must be fast and usually either defer most of their process completely or offload it to appropriate hardware.

Interrupts also have a hierarchy, like the following:

![[Pasted image 20240401233145.png|300]]

Here, `DPC` is the **Deferred Procedure Call**, and is just our good old software interrupts! The way you utilize them is that you submit a function pointer that would be asynchronously invoked in the interrupt context:
```C
void InterruptHandler( ) { 
	// deal with interrupt, IPL already ≥ 3 
	... 
	QueueDPC(MoreWork); 
	/* requests an asynchronous DPC interrupt */ 
} 

void DPCHandler( ... ) { 
	while(!Empty(DPCQueue)) { 
		Work = DeQueue(DPCQueue); 
		Work(); 
	} 
}
```

Linux handles this interrupts with a special kernel thread, so it is not really implemented in the interrupt context, since that kernel thread is scheduled like any other kernel thread.


# Storage Management

There are two different types of storage space:
- **Primary:** The physical memory, the RAM. Not considered a separate device, since it is directly addressable with a single level of indirection.
- **Secondary:** The disk/SSD, which is accessed with virtual memory only

The primary memory is considered to be much *much* smaller than the secondary storage, while being much faster. The main goal is to keep things as fast as possible, while allowing the applications to not care about how much memory they want (i.e. they could even use much more memory than the physical RAM allows them). This is only possible with virtual addresses and a lot of OS magic, that uses the primary and secondary storage in tandem.

How do we access secondary storage? Well, as you might remember, the **File System** was the main abstraction of the secondary storage, and that is still true. To be more precise, the kernel provides two suites of Syscalls that can be used to access the file system:
- **Sequential IO:** Which are `open`, `read/write` and `close`. These are operations where you open a byte stream to the file and read or right to it sequentially.
- **Block IO:** Which are `open`, `mmap` and `close`, which copy an entire chunk of a file into memory in one go. Once this is done, you can manipulate the file in memory (i.e. read or write to it), and then once you are done, call `close` to signal the OS to put all of your work back on the disk.

We have seen the first step to access virtual memory addresses, and that was using page tables. All virtual addresses *MUST* resolve into the page table, if not, a **Page Fault** has occurred. A page fault, is **actually a trap**, that signals the kernel to *try* and resolve the page fault.
Either the kernel succeeds and resolves the page fault, or it cannot (like for example, referencing memory location 0, i.e. NULL), the page fault turns into a **Segmentation Fault** and the OS will by default, kill the process that caused it.

## Address Space Representation For Processes

In a PCB, the kernel uses internal data structures to track the current use of an address space like the following:

![[Pasted image 20230328174601.png]]

Here, the PCB has an **address description field**, which points to a ***doubly-linked list*** of structures which we call `as_region` for now (Address Space Region/Segment). These data structure contain:
- The start and ending address
- File access:
	- `read` meaning that it can be read by a program
	- `write` meaning that it can be modified by a program
	- `execute` meaning that a program can point `EIP` to the start of this block
- private/shared usage, meaning whether or not it's possible for this exact block to exist in another PCB address description field. More precisely, it means whether or not we want to allow changes to be reflected in other processes executed on the same file.

The actual data in each part, can either be created by the process in terms of Text, Data, BSS and stack segments, or they can come from a disk, which means that they point to a file object provided by the file system.

>[!NOTE]
>In the above figure, from left to right, the sections are Text, Data, BSS, some random opened file and the Stack.

>[!TODO] For The Future ...
>If you get a Page Fault, you need to resolve it by walking through the above address space description. This is the main topic of Kernel Assignment 3.

>[!IMPORTANT] Not On Page Faults
>Under the following circumstances, where a program:
>- Accesses a virtual address not in it's primary address space
>- The translation of it's requested address isn't in the hardware map
>  
>A Page Fault happens, and wakes up the OS to come and resolve it.

## The File System(s)

The entity known as the **File System** is the system that manages objects on the disk. To be precise though, there are actually two meanings for this word:
- The specific layout of data on disk
- The way to actually access data on disk

As such, we divide the two concepts into **independent** and **dependent** file systems:

- **Independent File System** is the entity that implements the **File** abstraction as we see in all programming languages and IO programming. On UNIX, this entity is called the **Virtual File System**, in Windows, it is called the **IO Manager**.
- **Dependent File System** is the entity (part hardware, part software) that actually manages bus cycles and data layouts on the disk. 
	- In UNIX, this is called the **Actual File System (AFS)**, which comes in many forms (`ext1`, `ext2`, ...)
	- In Windows, it is called (regrettably) the **File System**, which supports things like `FAT32`, `NTFS` and others.

The two entities above are tied together in the kernel through kernel data structures, most prominently in the **System File Table**:

![[Pasted image 20230328193743.png]]

We've seen the above figure before, but let's go into a bit more detail:
- The middle table, the **System File Table** or **File Object Table**, is the independent part of the file system, that abstracts away the Actual File System, which might be hard to use. As you see, this table contains:
	- A *Reference Count*, which counts how many people are using this entry
	- An *Access Mode*, which is the `r`, `w` and `x` flags that we saw before
	- The *File Position*, which is the cursor used to interact with the file in a byte stream
	- And finally, and most importantly, the **I-Node** pointer, which points to an entry in the AFS table.
	  The I-Node is the data structure that is created and maintained by the AFS, and the kernel mostly relies on only the interface provided by the AFS to manipulate them, as such, only pointers to such data structures are used by the kernel.
	  As such, I-Nodes form the boundary between the two file system entities.
- Each process maintains it's own file table, which points to entries in the kernel file table, allowing the kernel to keep track of what everyone is doing.

The I-Node pointer, points to a series of functions in the AFS. What are these?
Well, as we have seen when discussing drivers, they are pointers to an *array of function pointers*, which don't have an implementation in the actual kernel, but rather in the I-Nodes pointed to by the file object.

Thus, in a simple manner, File Objects look something like this in C++:
```c++
class FileObject {
	unsigned int file_pos;
	unsigned short refcount, access;
	...
	virtual int create(const char *, int, FileObject**);
	virtual int read(int, void *, int);
	virtual int write(int, const void *, int);
}
```
In more suitable CS jargon, **Polymorphism** is used to tell the kernel what to do, without actually needing to implement these functions.
What about in pure C? As we said, it's just an array of function pointers:
```c
typedef struct {
	unsigned short refcount, access;
	unsigned int file_pos;
	...
	void **file_ops; /* Pointer to an array of function pointers */
} FileObject;
```
Not only does this make the implementation of kernel much MUCH cleaner, it also allows the code to work with pretty much any other AFS!

Alongside these, there is also a **File System Cache**, which keeps copies of recently used files to make disk access *feel* faster.
