# Execution Contexts

Until now, we talked about the OS only in terms of abstractions (i.e. threads, processes, etc.). We are yet to discuss it's implementation, and in fact, not yet ready for it!

We are going to spend some time discussing things "in between", things that are mostly decided by how we access hardware and such.
We'll start with the most important hardware, the CPU!

## Context Switching

Context Switching is the act of leaving a process, going to handle another process, and coming back to the first process *exactly from where we left off*. This is quite an important operation in the OS, and requires to be part abstraction and part implementation right from the start.

We have 4 "flavors" of context switching:
- Procedures
- Threads and Coroutines
- Syscalls
- Interrupts

Before we go further, what really "is" this context that we are throwing around?

In brief, the context is just the state of the thread/process, meaning:
- CPU register values (EIP, ESP, etc.)
- Stack
- Open files/sockets
- etc.

In brief, anything that changes how a code is executed is considered part of it's context. The stack ends up being the more complicated of these.

### Procedure Context Switching

The context of procedures (i.e. functions) is:
- Global variables
- CPU registers
- Local variables
- Arguments

Global variables have fixed addresses, as do CPU registers, but local variables and arguments live in the stack frame, and their address isn't immediately obvious.

![[Pasted image 20230227005912.png]]

Above you can see the stack frame used in Intel x86 CPUs.

Note where `EBP` and `ESP` point:

![[Pasted image 20230227010154.png]]

The `EBP`, also known as the **base (frame) pointer**, manifests in two ways:
1. It is actually a register in the CPU, pointing to the EBP value in the current stack frame (i.e. the EBP outside the stack frame in the figure above)
2. Inside the stack frames, EBP points to the stack frame directly *below*, thus, the stack is just a singly-linked list where the head is stored in the EBP **register** in the CPU, and the "next" pointers are stored in the EBP **regions** in the stack.

There is also the `EAX`, which will contain the return value of the function corresponding to the current stack frame. This is always a 32 bit value, thus you HAVE to either return a 32 bit value, or a pointer.

>[!NOTE]
>Most things in the stack are technically optional, meaning that they may not appear in certain cases:
>- Local variables may not exist at all in a function
>- Arguments may also not exist for a function
>- If the function performs no operations, we probably won't even have any registers to save
>- EVEN `EBP` is technically optional! If no arguments or local variables or registers exist, the EBP in the CPU will just point directly to the stack frame below, since the only value of importance (i.e. `EIP`) is already in the CPU.
> 
>Thus, the only mandatory part of the stack that appears for all things, is `EIP`. You MUST know where you should go back to.
>

Now that we know the semantics of the stack, let's actually discuss *how* functions are called. The general term we use for a function is **subroutine** in OS jargon, and the act of connecting calls between subroutines is called **subroutine linkage**.

Before we discuss the semantics of this, we should specify *who* puts what in a stack frame. Once again, there are **arguments, `EIP`, `EBP`, saved registers, local variables**. 
- Arguments are set by the <u>caller</u> , indeed, who else would know what the arguments of a function call are except the caller?
- `EIP` is copied into the stack frame by the <u>caller</u> using the `call` machine instruction.
- `EBP` is copied into the stack frame by the <u>callee</u>, indeed who else would know what the value of that is?
- Registers are saved by the <u>callee</u>. The logic is that if the caller had used some registers before calling the subroutine, upon the return of the subroutine, the value of the registers should naturally remain unaltered, thus, the callee will immediately copy all registers in the CPU into the stack frame before it attempts to run any of it's own code, and only then, it will proceed to use registers and alter their current values.
- Local variables, of course, are also *allocated* by the <u>callee</u>. We say allocated to emphasize that this is the **only** thing that the callee does, it **does not initialize them to any particular value**.

So, for example, what's the stack frame of the following?
```c
void func() {prtinf("Hi!\n")}
```
- Does this have arguments? Nope
- Does this have local variables? Nope
- Does this use any registers? Not by itself, `printf` does, but `printf` is a separate function call that needs it's own stack frame!
- Since no arguments and local variables exist, we just have the `EIP` to put in the stack.

>[!WARNING] A Little Note of Caution
>In the real world, there are *other things* between individual stack frames. 
>In this class, we don't care about that.

#### An Example

To see how a stack frame is created, we need to look at the machine instructions generated by compiling a real C code. To see an example, consider the following simple snippet of code:
```c
int main() {
	int i, a;
	i = sub(a, 1)
	return 0;
}
```
Allow us to remind you that C code starts by executing a **startup procedure**, which exists solely to call the `main` function with arguments (if there is any). Thus, the first stack frame belongs to `main` itself.

The `main` has no arguments here, thus the first thing that is pushed into the stack, is the `EIP`, now we go into `main` and the rest of the stack frame is handled by the callee.
In the beginning, the stack looks like the following:

![[Pasted image 20230227234931.png]]

As you see, `EBP` points to where the call to `main` was made, and `ESP` points to the top of the previous stack frame (there isn't any yet, thus it is also the bottom of the stack).

The code compiled for `main` would be the following:
```
main:
	pushl %ebp
	movl %esp, %ebp
	pushl %esi
	pushl %edi
	subl $8, %esp
	
	pushl $1
	movl -12(%ebp), %eax
	pushl %eax
	call sub
	add $8, %esp
	movl %eax, -16(%ebp)

	addl $8, %esp
	movl $0, %eax
	popl %edi
	popl %esi
	movl %ebp, %esp
	popl %ebp
	ret
```
To refresh your memory about machine instructions:
- `pushl %a`: Push Long, long just means 32 bit, and thus this means push the 32 bit value of `a` into the stack. This is equivalent to a `movl` operation and decrementing ESP by 4 bytes.
- `popl %a`: Pop Long, which means to pop 4 bytes from the stack and store it in the register `a`. This also increments ESP by 4 bytes.
- `movl %a %b`: Move Long, copy the value in `a` into `b`
- `subl %a, %b`: Subtract the value of `a` from `b`
- `$n`: Just means the value of `n` as number (thus `$8` means the number 8 stored in some temporary register).
- `n(%r)`: Means value in memory address that is created by adding the constant `n` to the value of the register `r`. Note that `n` can also be negative.
- `call f`: Goes to the code block of `f`, <u>and moves the EIP by one instruction and pushes into the stack</u>. Thus when we return from the call, we are actually pointing to the next instruction.
- `ret`: The reverse of `call`, it increments ESP by 4 bytes and puts the value of EBP in EIP.

Alright, let's make sense of this code.
```
pushl %ebp
```
This means to copy the current value of `EBP` into stack, this just pushes the EBP value into stack. 

![[Pasted image 20230227235826.png]]

```
movl %esp, %ebp
```
This copies the value of ESP into EBP. Now both of these are pointing to the same place.

![[Pasted image 20230227235922.png]]

Now, EBP pointers link all stack frames. Since the current value stored in the EBP portion of stack, points down to the previous stack frame, and the current value of the EBP register in the CPU, points to the stack region of EBP; Thus we can traverse the stack frames as we wish.

The two following instruction are curious:
```
pushl %esi
pushl %edi
```
We are in the part where we should save registers, but what in `main` actually needs registers at this point? The `ESI` and `EDI` are *index registers*, we don't talk about them here, since they are Intel specific, just know that calling `main` changes their values, thus it makes sense to backup them in the stack frame.

Consider this just pure bureaucracy. It's not our concern.
Performing the two instructions, we now have the following stack schematic:

![[Pasted image 20230228000525.png]]

Now, we need to allocate space (and just **SPACE**) for local variables. How many do we have? Well there is only the two `int` values `a` and `i`, thus we have 2 local variables, each 4 bytes, thus we need 8 bytes of space.

>[!IMPORTANT] Order Of Push
>In general, when we read variables/arguments in order in code, the ***opposite*** order is used to push them into stack.
>This means that while `i` appears before `a`, the space for `a` is first allocated and then space of `i` (this means that `a` is pushed first, and only then `i`).
>Same holds for arguments, when calling `sub(x, y)`, we first push `y` and then we push `x`.

To clear space, we just move the `ESP` up by 8 bytes, which is the same as decrementing the ESP pointer by 8 (remember, stack is in the **bottom** of the address space, to go "up", you need to decrease the top pointer).

That is what the following does:
```
subl $8, %esp
```
![[Pasted image 20230228001042.png]]

The above is the stack frame and the size of each region in bytes. Now, we have a full stack frame and ESP points to the top of the stack! We are ready to transfer control to the function `sub`, and anything in `main`.

>[!NOTE] Note About GDB
>If you set a breakpoint right at the beginning of `main` for GDB, where does it actually exist in the generated machine code?
>
>Well, it can't be right at the beginning, how would we divert control to GDB and return, when the stack frame of main isn't even created??
>The answer as you might imagine is right after decrementing ESP. The compiler will insert a `goto` statement that goes into the space where the GDB code for breakpoints is stored.

Now, we start creating the stack frame of the function call to `sub`. First, we have arguments. The arguments here are `a` and the constant `1`. The order of push is the reverse of this, thus the constant 1 is pushed first, and then the value of `a`.
```
pushl $1
```
The value of `a` is stored in the stack frame for `main`, in the local variables section. For `a`, first note the following schematic:

![[Pasted image 20230228002701.png]]

Starting from the bottom of the stack frame of `main`, which we can track by using the value of the EBP register, we need to traverse up to reach where `a` is stored. Note that to read the value of `a`, we need to point to the "top" of `a`'s region in the address space. 
The Saved Registers portion of the space is 8 bytes long, and the portion for `a` is 4 bytes, thus we need to go up by 12 bytes to reach the beginning of `a`'s memory location, thus to push the value of `a`, we can do:
```
movl -12(%ebp), %eax
pushl %eax
```
Whew! 

Now, we call `sub`.
```
call sub
```
Now, the rest of the stack frame for `sub` will be created in the callee. Once we return, literarily **nothing** will be changed, the stack will look like the following:

![[Pasted image 20230228003423.png]]

Now, it's time to cleanup! We need to pop the argument portion of the stack frame for `sub`. To pop this portion, we just go down by the length of that portion, which is 8 bytes:
```
addl $8, %esp
```
Now, we need to **return the value from `sub`**. As we said, the return value is stored in the `EAX` register upon the completion of the call. Who should get that value in `main`? Well, here it's the local variable `i`! 
Using the same logic that we used for pushing `a` into the stack, we can now copy the value of  `EAX` into `i`:
```
movl %eax, -16(%ebp)
```
Now, we have reached the `return 0` instruction. 
Again, we must set the return value into `EAX`, thus:
```
movl $0, %eax
```
Now, we are in the Saved Register portion and we are about to return to the start routine. We should restore the values of the registers so that the startup routine isn't surprised. The only register we had were `ESI` and `EDI`, both index registers, thus we pop in the reverse of the order that they were pushed.
```
popl %edi
popl %esi
```
This is also just bureaucracy. It's not our concern.

Now, the stack look like this:

![[Pasted image 20230228005403.png]]

The default instruction generated is:
```
movl %ebp, %esp
```
Which mirrors the first instruction we had. Here, this does nothing, but there are examples where it does something. This is just essentially a no-op, there is no sense making the compiler more complex just to prevent a single useless instruction from generation.

Finally:
```
popl %ebp
```
Now, we are done. The stack frame is fully cleared, now we just return to the startup routine.
```
ret
```
And we are done. Let's do the same with `sub`. Here, we coded it like the following:
```c
int sub(int x, int y) {
	// Computes pow(x, y)
	int i;
	int result = 1;
	for (i = 0; i < y; i++)
		result *= x;
	return result;
}
```
Let's do it in one go, again, the stack looks like this:

![[Pasted image 20230228010535.png]]

The beginning  (and end) of the compiled code is exactly the same as before:
```
sub:
	pushl %ebp
	movl %esp, %ebp
	...
	movl %ebp, %esp
	popl %ebp
	ret
```
Doing the first two lines, the stack ends up like this:

![[Pasted image 20230228010739.png]]

We have no registers to backup, thus we just move to local variables. Here, the local variable `result` is initialized by the programmer explicitly to be 1. We should make sure we do that.
Also, the compiler will notice that immediately; next, we have a for loop that initializes the value of `i` to 0. This needs to be done before we move into the loop, thus the compiler also generates code to initialize `i` to 0.

Again, we push in reverse order, and `result` came before `i`, thus we first create space for them and then we give them values:
```
	subl $8 %esp
	movl $1 -4(%ebp)
	movl $0 -8(%ebp)
```
We really don't care about the loop, let's just ignore it.
Assuming the loop exits, we now are at `return result`, this is easy!
```
movl -4(%ebp), %eax
```
Now, after executing the command above, the stack looks like this:

![[Pasted image 20230228011635.png]]

Now, ESP and EBP don't match, thus the instruction `mov; %ebp, %esp` actually does something now!

Done!

### Thread And Coroutine Context Switching

Normally, threads are not aware of each other and run independently.
However, if you want to joggle the CPU between threads in the same CPU, there is actually a way to do that called **Coroutine Linkage**.

#### Thread Context

The context of a thread is:
- It's stack
- It's register state

When we put a thread to sleep, we want remember all registers (EIP, ESP, etc.) in the context of the thread (i.e. the Thread Control Block) so that it can be restored once the thread comes back.

Note that to **run** the thread, one needs only to copy the context of the thread into the CPU (namely, ESP and EIP). Thus, to differentiate between different contexts, we call the thread running inside the CPU as the **Current Thread**, you can just imagine it as a global variable containing a pointer to the TCB of the running thread.

##### Example: Sleeping When Calling `pthread_mutex_lock`

>[!WARNING] Important Note
>Everything we discuss from now on, is only for a single CPU machine!
>Things get much more complex when there are multiple CPUs.

Consider two threads `A` and `B`. At the moment, `A` is running but `B` is sleeping. Note that:
- When a thread first comes back to life, the TCB of the thread needs to be consistent, to make sure that the thread resumes execution from the right place.
- *During* the execution of the thread though, the TCB is never touched, as it is just too wasteful to update the TCB in the middle of execution, when we aren't sure thread switch is actually going to happen.

So, schematically, this is how it will look:

![[Pasted image 20230303232144.png]]

At the moment, thread `A` is the current thread, and thread `B` is waiting.
To switch control to thread `B`, one can do something like the following:
```c
void switch(thread_t *next_thread) {
	CurrentThread -> SP = SP;
	CurrentThread = next_thread;
	SP = CurrentThread -> SP;
	return;
}
```
Imagine that thread `A` called `pthread_mutex_lock` and went to sleep, since it didn't have the lock at the moment. Ideally, we want to switch the CPU to another thread before we go to sleep for reasons that we discussed at length before, thus, the function above would be needed.

So, what is it actually doing?
Note that whatever data we actually have in the TCB of thread `A`, it's **outdated**, as it does not mirror the code that thread `A` actually executed to get there. 

First, when `switch` gets called, a new stack frame is pushed into the stack, and thus we need to update EBP and ESP:
- EBP will point somewhere in the middle of `switch` stack frame
- ESP will point to the top of stack frame for `switch`

Now, to update the stack pointer in the TCB, we copy the current value of ESP into the `SP` field of the TCB by using `CurrentThread -> SP = SP`.

![[Pasted image 20230303234242.png]]

Next, we set current thread pointer to be the TCB of thread B.
Finally, we set ESP to be the stack pointer of the new TCB, effectively switching the stack that the CPU will work on.

Now, here is the tricky part, buckle up!
The next instruction that we should run is `return`, but on who?

Remember that `return` essentially:
- Pops the top stack frame, moving ESP down by one whole stack frame
- Hops on the EBP value to get to the middle of the stack frame below
- Sets return value (which we don't have any)

Now, at the moment, the schematic of the CPU and stack figures is kind of a mess ..

![[Pasted image 20230303235149.png]]

As you can see, half of the CPU is pointing into stack A and the other into B. 
Now, since ESP points into the stack of thread B, what we actually do, is to pop a stack frame in `B` instead of `A`, what actually is that stack frame?

Turns out, it's actually the same `switch` function! How could have thread B went to sleep in the first place? Well, it probably executed the same `switch` function!
That's right, the actual `switch` function execution of thread B just finished now!! What we actually executed was technically the `return` statement of that function, not the one in thread A! Weird ..

Upon the execution of the `return` statement, EBP is also updated by looking at the middle of the popped stack frame (i.e. thread B's `switch` stack frame). 
We are finally **actually** in thread B! =)

Also, notice that the SP in thread B context field actually points to invalid memory now! That's what we mean when we say that TCB data after execution of that thread resumes, is inconsistent.

![[Pasted image 20230304000050.png]]

>[!IMPORTANT]
>What we actually see above for `switch` is just pseudo code, of course, C does not give you anything to manipulate the stack.
>
>What we actually use is a bunch of hand written machine code similar to what we saw in the previous section.
>
>The code above also is just incomplete, there are a few things that we haven't handled. We'll return to this in the future.

#### System Call Context Switching

A system call requires you to transfer control from yourself (probably user space code) to the kernel, and then back.
Note that no **thread switching** happens for this, as far as this class is concerned, system calls change the thread's "personality", it's still the same thread, but with extra privileges.

Context switching however would still be needed, so that we can return to exactly the place we were.

>[!NOTE]
>Most systems have separate stacks for user space and kernel space.
>Everything we discussed until now, has been in the stack that was for the user space. System calls use the stack for the kernel, and the user space is not allowed to access that stack directly, it's too stupid to do that and can crash the kernel.
>
>Part of changing the thread "personality" is actually switching to the kernel stack. The code that is executed in the kernel space though, is really essentially the same as the user space in nature.

We have seen that, in order to execute Syscalls, we need to trap into the kernel with a `trap` instruction (i.e. a software interrupt). To see this in action, let us consider a simple example:

##### Example: `write`

All Syscalls are essentially a thin wrapper around a `trap` instruction and some function that needs to be executed. So consider the following:
```c
void prog() {
	// Something ...
	write(fd, buffer, size);
	// Something
}
```
Now, from a really high level, `write` is:
```c
int write() {
	// Something ...
	// Setup for trap
	trap(write_code);
	// Teardown for trap
	// Something ...
}
```
This, is still executed in the user space, until the `trap` instruction is processed and we enter the kernel. Thus, when we execute `prog` and start calling `write`, the user space stack would look like this:

![[Pasted image 20230304015110.png]]

The C code inside the kernel is an **interrupt handler**. Here, the particular interrupt corresponds to the one for System Calls, which would look something like this:
```c
int intr_handler(intr_code) {
	// Seomthing ...
	if (intr_code == SYSCALL)
		syscall_handler();
	// Something ...
}
```
Here, `syscall_handler` exists specifically to handle System Calls, and here, the kernel uses the code provided to the `trap` instruction to distinguish what System Call was done (which we denote here with `write_code` for the Syscall `write`):
```c
int syscall_handler(trap_code) {
	// Something ...
	if (trap_code == write_code)
		write_handler();
	// Something ...
}
```
This, is just ordinary C code, and the kernel stack upon the execution of `syscall_handler` would look like this:

![[Pasted image 20230304015555.png]]

For Intel CPUs, there are 256 different trap codes, mapping to different System Calls. Upon executing `write_handler`, it's frame, along it's particular arguments will also be pushed into the kernel stack, and once we return to the System Call handler, we return to the user space when the kernel stack is empty.

This is the part where actual context switching takes place, as it's particularly important to handle return codes for different System Calls in the user space. That's the part we are yet to know, and we will discuss this in the next lecture. For now, just know that other than that, nothing executed here is out of the ordinary for C code.