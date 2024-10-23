# Booting

Booting is the process of starting the OS from 0, right after you power on your machine.
The process at least, involves copying the OS code from disk/persistent drive into the memory. To do this, it would require you to either:
- Boot manually by telling the machine what to do
- Use another machine to boot the machine
- Use another persistent drive to perform some operation to boot (like a USB stick)
- Leave it to hardware (which is what we do)

In early OS design, all devices and drivers were statically linked to the OS (i.e. hardcoded).
The problem with this, is that it needs to **assume** what kind of hardware or device will connect to the OS, thus if you add any weird thing to the OS, it will break!

Now, the OS actually *probes* to find what devices it should service, and based on what it sees, it will load specific drivers into the memory.
Now, it's even faster, since the part where we load the drivers is done dynamically **after** the OS actually boots, that's why we no longer need to leave and grab a cup of coffee each time we boot an OS.

## Modern Booting

These days, there always exists a specific chip on each machine called the **BIOS**, or **Basic Input-Output System**, which contains the most basic drivers required to run and operate basic I/O devices.

This I/O functionality is paramount, since:
- If something breaks, in the worst case we can bring some engineer to actually connect to the device and fix things by hand, as long as some basic I/O still exists.
- It allows for putting important configuration data on volatile memory (e.g. the RAM) 
- Interface with other required chips (such as CMOS, a chip used for booting Windows machines that specifies boot order of devices), and if needed, save them into non-volatile memory. For example, into the non-volatile RAM (NVRAM).
- Do self checks after startup (called **power-on tests** or **POST**). 
  For example, finding where the memory is and how much of it there is. This is also the place where we locate the **boot-loader**, which is tiny tiny operating system and helps with powering up the real OS.
  In case we also have multiple OSes on the same machine (e.g. dual booting), we need to also configure the boot-loader to ask us which one we should boot into.

The POST phase is particularly important in configuring the machine for OS operation. In most systems, it looks like this:

- On power-on, CPU executes BIOS code located in the last 64 KB of the first MB of the address space.
- We start from location `0xf0000`, and then we execute code hard written in the CPU that causes us to jump to the POST code.
- POST now initializes the hardware and counts memory locations and test them.
- We now find a boot device, usually we also ask the CMOS to tell us in which order we should boot things.
- Finally, we load a program called the **Master Boot Record (MBR)** which executes OS specific code to bring that OS up.

Schematically, it looks like this:

![[Pasted image 20230304172710.png]]

As you see, MBR is just 512 bytes long.
- The first 64 bytes describe how the hard drive has been partitioned (i.e. where each partition starts and ends, and what type they are)
- The last 2 bytes are called **Magic Numbers**, which upon the reformat of the hard drive, this part is set to a particular pattern to signal to the boot program that this device cannot be used for booting.
- The rest is boot program, which the bootloader loads into the memory and executes to start the operating system.

The MBR allows for the coexistence of multiple operating systems (as long as care has been taken by the user to install them in partitions not already used by another OS). For MS-DOS boot programs, a partition in the first 64 bytes is labeled as the *active partition*. The MS-DOS boot program looks into this particular partition and loads the first sector of it, which contains a special, OS specific boot loader known as the **Volume Boot Program**. The boot loader then context switches to that program, and the Volume Boot Program then brings up the operating system by itself, and transfer control to it.

## Linux Booting

In particular for Linux, the Boot Program would be one of the following:

- A Linux specific code called **Linux Loader** or *LILO*, which should be reconfigured if certain things in the kernel change.
- The more common **Grand Unified Boot Manager** or GRUB, which works with different OSes and file systems, and can actually find different OS images just by looking through a given system path name.

We use GRUB nowadays, both for Linux and Windows. If you ever used a dual-boot machine, or did it yourself, you probably are familiar with how it works, or might have seen it's messages or prompts.

After the boot program finishes its job, it's the kernel's job to configure itself now. In Linux:

**Note:** All of the following needs hardware specific assembly code!

- **Decompress Kernel:** We unfold the kernel code into memory by clearing the stack, clearing the BSS and transferring control to the loaded program.
- **Create Process 0:** Process 0 or the **Idle Process** in some jargons, is the first process that we create to handle basic things, like setting up initial page tables and such, and some basic paging and address translation (yes, even now, we need virtual memory addresses!!). 
  Once we turn on address translation, we can never go back, thus **everything that the kernel does is now done with virtual addresses, it can't use physical address anymore!**
  Don't confuse this with the INIT process, that's a different thing!
- **Create Process 1:** Process 1 is the **INIT** process that we mentioned, the ancestor of all processes. We now execute assembly code to initialize registers, and invoke the scheduler (which is equivalent to calling the `switch` function and pass the CPU to another process).
  After this is done, we can *finally* execute real C code.

Our kernel assignments actually start after the second step above finishes, right after IDLE is created, CPU is passed on to the kernel code by using `switch`.

>[!NOTE] Beyond BIOS
>BIOS was developed for 16-bit x86 long ago, and it's not readily available or extensible to other architectures. 
>Today, open firmware exists for different architectures developed by different vendors like Sun and such. The most widely used version is Intel's **Extensible Firmware Interface** or EFI. All of these use bytecode in order to be compatible with almost anything.

# Chapter 4: OS Design

We will now discuss a bit of general kernel design for OSes.

The main functionality of a general purpose OS is:
- Processes
- Threads
- File System
- Networking APIs
- User Interface and I/O
- File Ownership Management

>[!NOTE] Performance vs Modularity
>Ideally, we want ***fast*** and ***modular*** operating systems.
>We want fast OSes so we can run multiple programs better, and we want modular OSes so we can understand what they do and hack them based on our needs.
>
>Turns out, these two go very much against each other!
>If you want fast OSes, you should minimize the number of operations required to achieve necessary functionalities, and if you want modular OSes, you must isolate different parts of functions into modules, and that increase stack management complexity just to track what the OS is actually doing!

The simplest form of a general purpose OS will have to support the following:
- Terminal display/keyboard support for I/O
- Network API and interface management
- Processor interface
- Primary storage management (i.e. RAM and virtual memory)
- Secondary storage management (i.e. Disk and file system)
- An "Alarm Clock", that prevents threads from hogging the CPU for too long.

The last one might be weird ..
It is worth noting that what we are discussing is called a **Preemptive Kernel**, meaning that it allows for threads to tackle into each other and borrow the CPU from whatever thread was running in the CPU. This requires schedulers to manage.

There are non-preemptive OSes out there, we DO NOT discuss them anywhere in this class, HOWEVER, the "weenix" kernel which is our assignment is actually non-preemptive! So if your thread does not give up the CPU willingly, your kernel will freeze!

## OS Components

![[Pasted image 20230304190630.png]]

You can see a high level view of the OS above.
We are now, particularly interested in process management (which is the goal of kernel assignment 1).

>[!IMPORTANT]
>The Weenix kernel implements only **one** thread per process!

### Processes and File System

The purpose of a **Process** is to provide an abstraction of memory, and thus the first thing that provides the context of a process is it's **Address Space**, but beyond that, processes also have:
- A list of **threads** that they are using
- A list of opened **Files** that they are using

We will keep on using the address space abstraction that we used until now (i.e. address space is a collection of contiguous addresses that contain "similar stuff", like TEXT, BSS, stack, etc.)

So first question, **How do you initialize the address space?**. Unix does it in two steps:
- Make a copy of the address space with `fork`
- Copy content of the real process to the new address space with `exec`

This is quite wasteful, since we really don't need to copy the entire thing from the beginning. Parts of the address space of the original process and the new one can be *shared* between them. Which ones?
- The TEXT segment is read-only always, so no problem with sharing that
- Data segment? Well, another process probably shouldn't see that, as it violates process isolation
- Stack segment? Absolutely not! That's as private as a part as it gets.

We can share parts of the data segment that we are *SURE* cannot be modified, but that does not look that easy, does it?

So what do we do?
Well, it turns out that we can just let `fork` copy the *page table* instead of the whole memory! We do indeed technically copy the address space if we are using the exact same memory pointers after all!

So, `fork` after it's execution will produce something like this:

![[Pasted image 20230327021403.png]]

So instead of copying gigantic blocks of memory, we just copy *pointers* to gigantic blocks of memory. Of course, we still need to think something about the read-write pages.

This brings us to `exec`, what should we do now?
Well, to wipe out the address space, we just set all access bits to `NULL` and that's it! Now, if we want to run a program, we use the buddy system to allocate memory for the text and data segments of our compiled program and then copy them into our address space and execute them. Done!

Indeed, this works, but it's also *slow*, not to mention that you'll have to do this for *every* execution of your program, shouldn't there be a better way?

Enter **Memory Maps**.

In order to make `fork` efficient, we should share as much as possible and defer work as much as possible. To this end, we use a **Page Table** data structure for all processes (this is technically part of the PCB of that process). In this page table, we note what permissions each page has, what type is it, and what the physical address is (in the actual implementation, we don't have the last one, we use virtual memory mapping to deal with that).

With this, we achieve the figure as we saw above, but we still have the issue of *when* to start allocating new pages to the child.
We can't do it right away (or at least we shouldn't), if we use the buddy system to get pages for the child, we are going to look really stupid when the child calls `exec` and wipes away all the address space!
On the other hand, we might have to do *something*, take the data segment for example. Unlike the text segment which  read-only for both parent and child and thus can be shared directly (this is called **Shared Mapping**), we need to give the parent and child the same initial value of the data segment, but let them write to it independently (this is what we call **Private Mapping**).

So what to do?
We can't mark these pages read-write, it won't work, and we shouldn't really allocate them directly to the child right away.
The solution, **Copy-on-Write**

### Copy-on-Write (COW)

Copy-on-Write is an optimization. It defers the allocation of actual pages until we are sure the child is attempting to modify a page in a private map. 

But how do we actually implement it?
We will come to the solution much later, but in brief:
1. Do private mapping but marking the page table entry of the child as **read-only**, keep the parent's entry untouched.
2. When the child attempts to modify the page, a **Page Fault** happens. This causes us to trap into the kernel. When that happens, we can see what exactly happened when the child called.
   If this was an illegal memory access, then we will kill the program. In this case though, the kernel sees that this was a modification to a private mapping that is still read-only, so NOW it actually allocates a page to it!
3. The kernel will modify the page table entry to be read-write, and then returns to the child and asks them repeat their previous line of code (i.e. some modification to the address in a private mapping)

For heap allocation, the same technique is used as well. One difference though is that the pages on the heap are **initialized to all zeros**. These page are brought into the memory of the process as an **Anonymous Mapping**.

It is also possible to modify files like this. We can bring an entire file directly into memory (see `mmap` Syscall). Mapping of files can also be private or share, read-only or read-write.
In summary


| Mapping Object | Mapping Type | Semantics                                                                                                                                         |
| -------------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| Files          | Shared       | **R/O:** Can be shared directly. Writing will cause a SEGFAULT.<br>**R/W:** Changes are seen among all processes, and can go through to the disk. |
| Files          | Private      | **R/O:** Wiring will cause SEGFAULT.<br>**R/W:** Must use Copy-on-Write. Changes will NOT go to the disk (that is what private means!)            |
| Anonymous      | Shared       | **R/O:** Like with files<br>**R/W:** Like with files, but change only live in the mrmory.                                                         |
| Anonymous      | Private      | Just like files                                                                                                                                   |

>[!IMPORTANT]
>Not all files or object an be mapped into the memory, or it may not make sense.
>This mostly makes sense for **Blocked IO** objects, whose natural unit is a page worth of data.
>Some things are **Sequential**, like keystrokes!

