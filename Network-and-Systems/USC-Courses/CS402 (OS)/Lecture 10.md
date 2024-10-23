# Dynamic Storage Allocation (cont.)

## First-Fit Implementation

For all storage allocation algorithms, we maintain a doubly linked list of `fblock` structures, which we call the **Free List**. The `fblock` structure differs between different algorithms, but at it's core, it consists of basically two fields:
- Pointer to previous and next structures in the list
- The size of the associated memory block of free addresses

We use a doubly linked list so that the operations of insertion and deletion is $O(1)$.

While we discussed how `malloc` works, we need to still discuss `free`. When we free a block of memory, we should take care to merge it with it's neighboring blocks of free memory like the following:

![[Pasted image 20230305161436.png]]

If we don't do this, future calls to `malloc` can fail for no particular reason at all! This is called **coalescing**, and to do this, we need to maintain the size of the blocks above and below the block being freed, and whether or not they are allocated or freed.

One solution here is **Boundary Tags**, which assigns different structures for allocated and free blocks like the following:

![[Pasted image 20230305161718.png]]

Here, for allocated block, the `size` field would be a 4 byte field (on a 32 bit machine) which holds the size of the ***whole*** block (i.e. the gray area plus **both** the `size` fields). We should maintain the `size` field both at the end and the beginning of the memory block.

### Example

Let's see a detailed example. Here, we use a structure that is a bit different compared to what we saw above.

![[Pasted image 20230305162439.png]]

We are going to be a bit wasteful with this way of management, but it's nice for understanding how this works.

Here:
- For **In-Use Blocks**, we set `in-use` to be 1, to note that the block of memory in between is for a user program, we shouldn't touch it.
- For **Free Blocks**, we set `in-use` to 0, and maintain `next` and `prev` pointers. The contents of the memory segment itself are considered garbage.
- For **Free-List**, if it is uninitialized, it is maintained in the BSS segment, and once it is initialized, we maintain it in the data segment of memory.

In the beginning, the entire heap segment of memory is a single free block maintained in the free list. Once we call `malloc` a few times, we segment it into different portions.

For example, to call `malloc`, assume we have 60584 bytes (i.e. `0x0000eca8`), assuming that the heap starts at the address `0xfedcba98`, the structures will look like this:

![[Pasted image 20230305162938.png]]

The `head` and `tail` right now point to the same place (namely `0xfedcba98`) and `prev` and `next` are both `NULL` (i.e. `0x00000000`).

Let's call `malloc(100)`, doing this, would mean that we want a memory portion of 100, and since we have 16 bytes of configuration data for allocated blocks (i.e. `in-use` and `size`, times two, since they are at the top and bottom, each 4 bytes on a 32 bit machine), we reduce the heap size by $100 + 16 = 116$ bytes, thus the remaining free block size is $60584 - 116 = 60468$, which is equal to `0x0000ec34`.

Schematically, we end up with this:

![[Pasted image 20230305163657.png]]

As you can see, the address of the actual free block is the blue part, starting with the address `0xfedcbaa0`, which is the return value of `malloc`. Also please note that the value of the `size` field is 116 instead of 100 (i.e. `0x00000074`).

You should also change `head` and `tail` accordingly.

If we call `malloc` consecutively, we end up with contiguous blocks of allocated memories like this:

![[Pasted image 20230305164039.png]]

Now, let's call `free`, we pass an address like `y` by calling `free(y)`, we know (assuming that the application has not violated the contracts we mentioned and corrupted the memory):

- Starting from `y`, the previous 8 bytes are `in-use` and `size`, thus if we look at the word in `y - 8`, we *SHOULD* see that it is `0x00000001`, since this is an allocated block. If it isn't, we have memory corruption and we now have segmentation fault!
  Assuming things are ok, we can see the length of allocated memory by examining `y - 4` and so we have identified how much memory we should free!
- Once we have this data, we should start to coalesce the memory block. To do this, assume the value in `y - 4` is `z`.
  The value in `y - 16` would be the `in-use` field of the block right above us, and `y - 8 + z` would be the `in-use` field of the block right below us.
  Once we know this, we can start coalescing.

For example, let's assume we are like this:

![[Pasted image 20230305164632.png]]

The previous block is free and next block is allocated. Since we are calling free, we set the `in-use` fields of the current block to be 0 (i.e. set `y - 8` and `y - 16 + z` to 0). To merge with the previous block, we only need to set it's `size` field and our bottom `size` field to be `z` plus the size of the block above.

To find the size of the block above (i.e. `w`), we need only look at `y - 12` for `w`. Once we have that, we set `y - 4 - W` and `y - 12 + z` to be `z + w` and we are done.

>[!IMPORTANT]
>Note that we didn't need to touch the previous and next pointers here!
>That probably won't always be the case.

![[Pasted image 20230305165053.png]]

All we changed, were 3 fields. That is the value of $O(1)$ complexity.

To merge with the next block:

![[Pasted image 20230305165138.png]]

Most of the arithmetic is the same as before, but there is one big difference. Now, the `prev` and `next` fields are incorrect after merging. 

First, we need to copy the `prev` and `next` fields of the block right below us to `y`, since we need to point to the correct positions now that we are a free block, not an allocated block. So we need to change 5 fields (technically 4, since 2 of them can be done at the same time with `memcpy`), are we done?

Well, no!
There is still the `next` pointer of the **previous free block**, which we don't actually see in the figure, but we can jump to it by following the address pointed to by the `prev` field. We need to do same for the `prev` field in the free block right after us.
So we are changing 6 fields!

If both the previous and next blocks are free, you can check that we also need only change 4 fields and still, the operation remains $O(1)$.

Weirdly enough, the worst case is the case where both the previous and next block are actually already allocated, and we have no way to merge the blocks. We must insert the freed block into the right place in the free list, and that requires an iteration over the list, thus in worst case, free is $O(n)$ complexity.

The same goes for `malloc`, it is also in worst case $O(n)$.

>[!NOTE]
>The memory allocator does not check whether or not the data structures are consistent. Thus, in the presence of a memory corruption bug, the **entire memory allocator can break.**
>
>Should we worry about that?
>Well, no! Fixing this is way too slow, since we have to check whether or not the structures actually exist in the free list and that is just too slow. The application using the memory allocator needs to stick to it's contract with the memory allocator and never go beyond where it is allowed to.
>
>You can probably see why memory corruption bugs are so difficult to detect now! You can mess up once, go about your life for many many operations and only detect the problem when performing some random operation at some random point in time!

## Kernel Performance

It turns out, since first fit `malloc` and `free` are *sometimes* $O(n)$ and *most* of the time $O(1)$, we ***cannot*** use them in the kernel.
The kernel requires that all operations have predictable timing so that it can handle asynchronous tasks without breaking itself, thus **first-fit is NOT acceptable for kernel memory allocation**.

By "Kernel Memory Allocation", we mean things like allocating PCBs or TCBs, signal handlers and whatnot.

The implementation that we shall now see for kernel memory allocation is called the **Buddy System**.
### Buddy System

The buddy system relies on the following constrains:

- All blocks get divided into two blocks of even sizes called "buddies". The minimum size of a buddy is defined as a **page** (usually 4 KB = 4096 Bytes).
- Only free buddies may merge into a single free buddy.

The first problem as you can see, is that this actually suffers from **internal fragmentation**, since if you try to allocate a single byte with `malloc(1)`, you *WILL* receive a block whose actual size is 4096 bytes, but you are only allowed to use 1 byte in it!

This is a space-time tradeoff, which is worth it if you are inside the kernel. While we end up wasting memory since we can only allocate blocks of size $2^i . P$ for a page size of $P$, it is paramount to be consistently fast in the kernel.

![[Pasted image 20230305191528.png]]

As you can see, this ends up in a binary tree list traversal and thus the complexity of traversing the list of buddies is $\log_2{n}$ for $n$ buddies, and this is good enough inside the kernel.

The buddy system maintains a list of lists. Each list, holds the address of buddies of a particular size in a doubly linked list (though, not circular). So there is a list of buddies of page size, a list for 2 times the page size, a list for 4 times, and so on.

Thus, `free[k]` will point to the first available block of size $P . 2^k$ by iterating the kth list in the buddy list for free buddy.

Each block will have `in-use` bit and `size` and `next` and `prev` as before.

#### Example

Take the following:

![[Pasted image 20230305192954.png]]

At first, there is only a single buddy of size 16 page, or $2^4 . P$ for page size of 4 kB. We denote an empty list with $\Omega$. The list `free` is the free list and is indexed with `k` from 0 to 4 (inclusive).

To allocate a block of two pages, we iterate over `free`. Do we have a block of 2 pages right now? No! since `free[1]` is empty. Do we have blocks of length 4 page to divide into two blocks of 2 pages? No! Since `free[2]` is also empty.

You can see where this is going, eventually we reach `free[4]`, divide it into two blocks of length 8 page, we put the second part in `free[3]` and keep dividing the first part and putting them in `free[2]`, `free[1]` until we have an address to our desired block of 2 pages.

If we now call for allocation of 4 pages, we have one ready in `free[2]` and thus we just allocate it and we are done.

Continuing, we end with something like the following:

![[Pasted image 20230305193739.png]]

### Slab Allocator

The Slab Allocator, or the **Array Allocator**, is an $O(1)$ allocator (yes, faster than the buddy system!).

To do this, we "cheat" a bit. In the kernel, we usually know how much memory we need to allocate for each operation (since for example, PCBs have known sizes). So, why not borrow a big chunk of memory from the buddy system, and break them down to slabs of required sizes?

Doing this, we'll end up with something like the following:

![[Pasted image 20230305194246.png]]

If you get a slab of 16 kB like above, we usually end up with leftover memory after we break the slab, but note that this is also **internal** fragmentation (even if the figure above makes it look like **external**).

For `malloc`, we just return the index of free block in the appropriate slab, and for `free`, we just mark the block as free and we are done!

This is only used for data structures used inside the kernel, like kernel stack, PCBs, file objects and other well defined memory sizes. Thus, this **is not a general solution**, we will still need the buddy system.

>[!FAQ] What About Kernel Data Structures Whose Size We Don't Know?
>First, we *do* have such things, like device drivers!
>
>The hardware engineer is the only person who knows how much we need, thus for these things we again need the buddy system.

# Linking and Loading

We saw previously how a code is converted to assembly code and its stack is created. It is important to note that the memory locations of the text and stack sections, don't really matter on their own.

You might ask, where does the location actually matter? Well, a lot of the time location **actually** really really matters! We were just looking at too simple examples. Here is a better one:
```c
int X = 6; 
int *aX = &X;
int main() {
	void subr(int);
	int Y = X;
	subr(Y);
	return 0;
}

void subr(int i) {
	printf("i = %d\n", i);
}
```
So, there are two main complications here:
- There are global variables, mainly `X`, which is stored in the Data segment, whose location matters.
- There is a function `printf` that *isn't here right now*. Where is it?

Even for `subr`, the location of that function is also variable, depending on where it is located in the Text segment, but the assembly code automatically assigns it a label so that we can track it.

Who is keeping track of these things? Making sure that things aren't in bad shape?

Well, the compiler is our first tool here. 
The compiler knows exactly how large the stack frame and any other variable is, it knows exactly how many bytes every object is. The compiler then gives each object  **temporary addresses** by just putting everything in a neat row and indexing the memory based on the length of each object in terms of bytes.

But, what about that `printf` business? Well, we need another program for this, dubbed the ***Linker***. The linker figures out where each object actually is, by using the temporary mapping that the compiler generates, and then mapping it to the actual virtual memory space. The things that the linker ultimately decides on are *functions, global variables,* and more.

So there is two steps until now:
- Ask the compiler to measure the length of every object and give things temporary addresses.
- Ask the linker to use the output of the compiler to figure out where everything is.

While step 1 seems intuitive enough, step 2 looks just weird. How do we even use the compiler output?

Before we go further, let us first say that we've done this a hundred times already in any warmup assignment or the kernel assignment, assuming that we were wise enough to use *separate compilation*, meaning that we compile related `.c` and `.h` files into `.o` files, and then *link* all of them into an executable file. The multiple compilation commands to produce `.o` files is equivalent to step 1 above, and the second step with the linker, is equivalent to the single link command to produce an executable. 

Getting back, what does the linker actually do?
Well, global variables should be defined *somewhere* after all, and the same goes for functions, which are either somewhere in the immediate address space of the `.o` files (like the `subr` above) or are included by using `#include`. The linker knows where libraries and code files are (libraries are common knowledge for all programs and code files should be given to the linker explicitly). Once we found these, we then **relocate** all these code segments into a big blob of address space, and use multiple `exec` Syscalls to allocate real memory for them. 

Finally, the linker, having all the pieces ready (assuming you haven't forgotten something!) assigns the address of functions by putting labels right where the assembly code of particular functions start, and linking global variables by just pouring everything right at the beginning of the code. If something remains unresolved in the end, the linker will print an error message and complain to the user.

So in brief, the linker just yanks everything from their previous temporary location assigned by the compiler in the `.o` files and ***relocates*** them into real memory spaces by just tracking things and using operating system Syscalls (mainly `exec`), and doing ***symbol resolution***, which is assigning correct addresses to things not immediately in the code space (i.e. the `printf` and `#include` stuff among others).

It is important to also mention the **Loader**. The loader is actually the program that executes the output of the linker. This is done by:
- "Unfolding" the rectangular blob outputted by the linker on the disk, into memory. Possibly filling some "holes" (we don't need the details).
- Putting `EIP` right at the start of this blob to begin program execution.

All that we mentioned is more specifically called **Static Linking and Loading**. There are loaders that do also relocate things like the linker does called *relocating loaders*. We'll discuss them much much later in the course. 

So, knowing this, what will the previous program really look like the in the real world? Well:
```c
/* main.c */
// We declare X as being outside of this code.
// This is where the compiler just gives X a temporary address.
extern int X;

int *aX = &X;
int main() {
	// This also gets a temporary address by the compiler for now!
	void subr(int);
	
	int Y = *aX;
	subr(Y);
	return 0;
}
```
And:
```c
/* subr.c */
#include <stdio.h>
int X;

void subr(int i) {
	printf("i = %d\n", i);
}
```
If you are not picky, you'll just do `gcc -o prog main.c subr.c` to get the executable `prog`. What happened to the separate compilation though??

Well, if you are not careful to do this yourself, GCC does it for you. When you run the command above what actually happens in the background is:
- GCC compiles `main.c` into `main.o` (we now have two "unresolved" addresses, `X` and `subr`, you can't run this output yet).
- GCC compiles `subr.c` into `subr.o` (this also will have an "unresolved" address, one for whatever `printf` is).
- The linker (i.e. `ld`) will now Frankenstein the two `.o`  files together, and also notice the `#include` in `subr.c` (remember, the compiling process really does not do anything with these statements, it just copies them with no change).
  The linker puts the real address of `printf` that it read in `stdio.h` over the temporary one assigned by the compiler and resolves `printf`. It also deduces the unresolved addresses of `subr` and `X` by looking into `subr.o` and put them in `main.o`. Finally, the result is brought into memory with `exec` and written as a full file on the disk with the name `prog`.

You will now run `prog` by using a loader and be done with this, which also uses a `exec` Syscall.

>[!NOTE] 
>Those `.o` files, are called **relocatable modules** to differentiate them from binaries (i.e. `prog` above).

>[!FAQ] What About Header Files?
>Local `.h` files are just copied right to the beginning of the code when the compiler sees them. There is no sense asking the linker to do trivial things.
>Other headers (like `stdio.h`) are left for the linker to resolve.

## A Closer Look At "Unresolved" Things

As we saw, in the relocatable modules outputted by the compiler, there re probably things that don't go anywhere most of the time, so how do you actually convert that to assembly instructions in the first place??

Let's see the output for the actual compilation of `main.c`. When needed, these files are post-fixed with `.S`, so `main.S` look like the following:

![[Pasted image 20230319191523.png]]

Whatever there is after `main`, is just the result of converting the actual code in `main.c` into assembly code without any regards for where `X` or `subr` is. The interesting and new parts is probably:
- The fact that there is now this weird `offset` column, what is that?
- What's with those directives that start with a dot sign??

Let's start with the second one. Those are assembly directives that signal to the linker where Data and the Text segments are.

In brief (a bit too brief truth be told!):

| Directive | Semantics                                                             |
| --------- | --------------------------------------------------------------------- |
| `.data`   | What comes after this, are initialized variables                      |
| `.globl`  | What comes after this, needs to be exported by all modules            |
| `.long`   | Allocate 4 bytes for the address that follows this symbol             |
| `.text`   | Restart the offset, we are entering the text segment                  |
| `.string` | What follows this, is a C string (i.e. a string terminated with `\0`) |
| `.comm`   | What follows this is the BSS segment                                  |

So with this in mind:
- The first line (starting with offset 0) just says that the data segment starts now.
- After that, we mark `aX` (remember that all symbols in the assembly code are just *addresses* not values) as global.
- We declare `aX` to have the temporary address of the current offset (which is still 0), and we mention that it is 4 bytes long in the next line. Thus after this, the offset goes up by 4.
- We are now finished with the Data segment, we signify it's end by saying `.text` and resetting the offset to 0.
- We declare the symbol `main` as a global variable. This is just C bureaucracy, since **everyone** needs to know where `main` is.
- We enter the `main` part and we start translating C to assembly as usual.

>[!FAQ] Should I Worry About The Fact That Offsets Are Overlapping?
>No! These are temporary addresses, the linker is wise enough to fix these later.

For `subr.S`:

![[Pasted image 20230319200239.png]]

In brief:
- The Data segment contains the variable `printfarg`, generated by the compiler. It is declared to be a string, and thus the offset must move forward by the length of the string. The length of `i = %d\n` is 7 (remember `\n` is just one character!), there is also `\0` in the end and thus we add 8 to the offset.
  The `printfarg` name is arbitrary, real compilers will probably say `var00xyz...` or another generic name.
- The variable `X` is not initialized, thus we declare it after `.comm` while also saying it's 4 bytes long and moving the offset by that much.

You can read the assembly code to see how these symbols are used (it's not needed for the class though). The linker will now use these files, and compactly pack these Text, Data and BSS segment into a neat object file with it's own Text, Data and BSS segments and give it to you as an executable binary.

You can actually inspect these outputs:
- Use `nm` to list symbols
- Use `objdump` to get detailed information

You'll use these for kernel 3!

In terms of semantics though, your `subr.o` is like this:

![[Pasted image 20230319201301.png]]

As you can see:
- The length of Data and BSS segments are explicitly mentioned with their contents.
- The text segment also has a specific size (note that there is 1 final byte not shown, as the size of instructions is 23 but the segment itself is 24).
- Some symbols are specifically mentioned to be "Undefined", like `printf` for the linker to resolve.
- Most importantly, there are "Relocation" instructions:
	- At offset 7 is the address of `printfarg` with length `4`. Why?
	  The instruction in question is actually `pushl $printfarg` at offset 6. The `pushl` instruction gets one byte and thus the address of `printfarg` will be loaded on offset 7, and since it's a 32 bit number, it will have a length of 4, so the offset after this instruction will be 7 + 4 = 11.
	- Same goes for instruction `call printf`  at offset 11, which references the address of `printf` at offset 12, since the `call` symbol adds 1 byte to the initial offset of 11.

These addresses are called **PC-Relative addresses**.

For main as well:

![[Pasted image 20230319202518.png]]

Again:
- Undefined reference for `X`, which is resolved in `subr.o` later by the Linker.
- Relocation instructions for `X`
- Declaring `subr` as undefined, and also again giving PC-relative addresses. Looking at the instructions, we'll see that the address for `aX` is 6 + 1 = 7, and the address for `subr` is 19 + 1 = 20.

So we are done, right??

No sadly, there are a few more headaches. Namely:
- What *actually* happened to `printf`??
- Who calls `main`??
- What if we use a Syscall somewhere?

First, `printf` has it's own object file as a C library, which again is a big blob of Text and Data and BSS segments like this:

![[Pasted image 20230319203612.png]]

As you see:
- The text segment (the actual code of `printf`) is pretty large, so let's ignore that, with the exception that we are calling a `write` Syscall which isn't in `printf.o`!
- There is a global variable `StandardFiles`. Taking an educated guess (since this is `printf`), you can probably see that they are the files `stderr`, `stdout` and `stdin`!

So what to do with `write`? Again, `write()` as a C function is just a thin wrapper around a trap machine instruction, so the actual object file of `write` which like `printf` is part of the standard C library, is rather anemic:

![[Pasted image 20230319203854.png]]

The only thing to notice is `errno` as a global variable, since if the write instruction fails, we should write to it! 

There is also the startup routine, which is the thing that calls `main` in the first place!

![[Pasted image 20230319204034.png]]

What does this do?
- It calls `main`
- Returns it's value
- Calls `exit` just like we called `write`

As you see, `main` is undefined! Who defines it?? ... YOU! (hopefully ...)

![[Pasted image 20230319204222.png]]

Letting of some details, the actual thing that gets put into the final executable `prog`, is like above. We just combine the Text and Data and BSS segments individually and pack them together into `prog`.

So what's up with the numbers?
The numbers are the start address of each segment. Since the buddy system gives us at least 4kB pages, we are seeing large numbers, but why not start from 0?

Well, that is deliberate, since we want all calls to the `NULL` pointer to fail with a Segmentation Fault, so there is instruction in that portion to handle this and call `exit` immediately. So from 0 to 4095 is reserved, thus `main` starts from 4096.

How fat was main's Text segment? 36, so add 36 to 4096 to get 4132, the start of the next Text segment for `subr` and so on. 
That said, you might have noticed a jump between the Text and Data segment. If things were really tightly packed, `aX` should have started from 16172 plus the size of `startup`, which is 36. That would be 16208, so why is it 16384??

It's not hard to see that this is because the Data segment and Text segment actually begin and end on different pages. The final byte of the Text segment is on the 4th page allocated by the buddy system, but the first byte of the Data segment for `aX` is on the 5th page, so it starts at 4 times 4096 which is 16384 as expected.

Why tough?
The reason is that **the Text segment MUST be read-only, whereas the Data and BSS segments don't have to**, and thus, they can't be on the same allocated page, as a page should explicitly be marked as read-only, read-write and so on.

![[Pasted image 20230319205608.png]]

Why keep the Text segment read-only? One can only imagine what sorts of disasters could happen if we let runtime code be changed by another program!! We'll have some really interesting viruses to deal with! So it only makes sense not to do that.

So, our program needs only about 16 kilobytes of memory plus a bit of stack space, but the process that will execute this program is going to have a 32 bit address space, a whole 4 GB of memory!!!

Surly, giving 4GB of memory to a program that only needs less than one megabyte would be ludicrous! 
Solution? You can probably see why we emphasized that everything a process has is just *virtual memory*. Indeed, the real memory used by the CPU is not *really* allocated by the compiler, it is just too stupid to do something so important, as it has no idea what *other* things are running. We need the kernel to help us here.

And indeed, the kernel has this thing called **Page Table**, which is a kernel data structure used specifically by the CPU. How it works is:
- The OS allocates pages of 4KB size using the Buddy System for programs (like ours above). 
  Remember that a "page" is just physical memory mapped to anywhere in a virtual memory. Thus regardless of how tightly packed the virtual memory space is, the physical memory can (and usually is) all over the place (and that's how we end up not wasting 4GB of data on everything).
- Every time we access the data pointed by virtual memory, the OS actually hops on to the physical memory associated with this address by using the page table, though the actual "address translation" is done by hardware, since doing this in the OS would be just too slow.

This has the down side of putting a bit of overhead on every write and read operation, but on the other hand, we can use physical addresses from literarily anywhere!! It makes working with hardware much much easier (and safer!).
