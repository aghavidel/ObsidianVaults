**Note:** We will skip the device driver part, it is not important

# Process And Thread General Abstraction

The main abstraction important for the OS about processes and threads is their associated Control Block (the PCB and the TCB), which are linked together like the following:

![[Pasted image 20240401205650.png|500]]

For PCB:
- The first chunk describes the pages owned by the process and how the virtual memory areas are organized (we'll see this later)
- The next chunk contains a pointer to the list of open files. This is an array of file descriptors.
- Next chunk is the list of threads owned by the process, which is an array of pointers to TCB structs (for our kernel assignments, we only have a single thread per process)
- The current state describes the process life cycle (i.e. is it dead or not?)

For TCB:
- We have the context, which includes the stack pointer, the registers, and whatnot
- We also have states for threads, similar to how we did for processes

The life cycle of a process is very simple. It is either in `RUN`, which means that it has resources allocated to it and it must keep them for now, or it is a `ZOMBIE`, which means that its resources must be returned to the kernel.

>[!NOTE]
>While extending the single thread-per-process implementation to multi-thread-per-process seems easy, there are some nuances.
>In particular, one source of difficulty is spawning processes using `fork`, what should we do in the second process threads?
>1. Should we just spawn a single thread in the new process?
>2. Should we spawn as many threads in the new process as the old one?
>The problem with the first approach for example is that if a thread holds a mutex (or multiple threads!), who should have it in the forked process?
>
>In POSIX, the solution is to just unlock everything and then fork. POSIX makes no assumption about how fork interacts with threads.

## Thread Lifecycle

Threads have a bit more to them:

![[Pasted image 20240401210458.png|500]]

In particular:
- A thread is spawned as `RUNNING`, and remains there as long as it has the CPU
- The scheduled at any point may decide to tell the thread to go to sleep, which is done by picking the thread TCB and putting it on the *run queue*. This is done solely by the scheduler and the thread will be `RUNNABLE`.
- A running thread may perform a blocking call that forces it to sleep, in that case it is in `WAITING`, and the scheduler has nothing to do with this. The sleeping mechanism though is the same.
  A thread may also voluntarily give up the CPU and put itself on the run queue (this is called *yielding*). This is how threads can be implemented in the user space without kernel support.
- A thread may call `pthread_exit` and then go to the `TERMINATED` state.

A thread in `TERMINATED` is basically a zombie, since its TCB is still there (it contains its ID, its return value and its stack). Normally, we would call `pthread_join` on it and free the TCB there, but we do have *detached threads* to worry about, which cannot be joined with.

For these, there are two solutions:
- Let the library maintain an internal reaper thread and only let that thread join with detached threads.
- Keep the detached thread IDs in a list somewhere, and let the scheduler occasionally free their TCB and pop the list when it sees fit.

# Thread Implementation Strategies

Now, we want to see how threads are *actually* implemented, and what design choices go into making them work.

One main question is this: **Where to actually implement the threads?**

We can implement them in user space, or kernel space:
- Kernel space is robust and strong, but creating a TCB requires a trap, which can be slow
- User space threads are very cheap and easy to spawn, but they cannot rely on the kernel for scheduling any more.

We have multiple strategies:

## One-Level Model

This is the simplest model, and basically it will implement threads and all aspects of it in the kernel space. So this includes:
- TCB allocation/deallocation
- Mutex implementation
- Synchronization and scheduling, etc.

Every time a user space call for thread creation comes in, it traps into the kernel, and there, the kernel will create a TCB for it and spawn a kernel space thread that will handle the user requests.

The problem here is performance. Traps are expensive, and so if you call `pthread_mutex_lock`, even if the lock is free, it can get relatively long, since there is a trap instruction to go into the kernel and see who owns the lock, this can degrade performance significantly.

This is not a concern when the lock is locked though, since that has to make a system call anyway and will cause the caller to go to sleep. For the unlocked mutex case this is important, since most Mutex locks are unlocked most of the time.

There are mitigation to this, for example, the **Native POSIX Thread Library (NPTL)** uses a form of user space mutex (called a *FUTEX*) to implement actual mutexes for the threads, and it is quite performant.

## Two-Level Model

In the two-level model, threads are implemented at least partially with the help of a user space library. This method has two flavors:

### The $N \times 1$ Model

In this model, the kernel will have a single thread, and user space can spawn as many threads as it likes. All thread allocation/deallocation and synchronization calls are implemented directly in the user space:

![[Pasted image 20240401212609.png|500]]

This is one of the easiest and earliest implementation of threading:
- It is fast
- It is easy to handle and understand

However, it has one major problem. If a user space thread uses a Syscall that could potentially block them (like `read` with a very large buffer), then the kernel will have to handle it directly, and since there is just one kernel thread, everyone else will freeze, since the kernel cannot answer to them! They must wait until that Syscall is finished!!

This is really bad, so there have been some ways to handle this. In particular, one solution was to make a specific kernel space form of every Syscall, like `real_read`, which can block and a form of it for users, like `read` that never blocks.
If the system notes that a call to `real_read` can block, then it would return an error like `EWOULDBLOCK` and return from `read` in the user space.

The biggest problem however is that these are really useless for multi-processors, since they cannot take advantage of parallelism among cores.

### The $M \times N$ Model

Here, there are multiple kernel threads for each process. As long as a process does not exhaust all the kernel threads it has, the problems with the previous implementation won't happen.

This still has problems though:
- We have not really resolved the problem. If we can have at most $N$ threads in the kernel for each process and if the user makes $N$ blocking calls, then all of these threads will freeze and no other user thread would be able to progress, whereas this may not make much sense at all.
- **Priority Inversion:** Threads can have priorities, which will influence the thread scheduler when it wants to put a thread on the run queue. However, user space scheduler and kernel space scheduler, generally cannot coordinate together efficiently, and this can cause weird problems, if a high priority user thread gets mapped to a low priority kernel thread.

## Mutex Implementation

We now discuss how to implement a correct Mutex. 
There are a few things to keep in mind, most importantly, interrupts!

![[Pasted image 20240401213733.png|400]]

There are 4 main cases to consider:
- A thread in the same CPU can preempt the current thread
- A thread in another CPU can preempt the current thread
- An interrupt handler on the same CPU can access a data structure we have
- An interrupt handler on another CPU can access a data structure we have

For now, we consider only **Straight Threads**, which are threads that run on a single CPU core and can be shielded from interrupts.

The first thing that we need is an implementation of context switching. To this end, we need to finish the `thread_switch` function that we started a while ago:
```C
void thread_switch( ) { 
	thread_t NextThread, OldCurrent; 
	NextThread = dequeue(RunQueue); 
	OldCurrent = CurrentThread; 
	CurrentThread = NextThread; 
	
	swapcontext(&OldCurrent->context, &NextThread->context); 
	// We’re now in the new thread’s context 
}
```
If the run queue is empty, then a call to this should do nothing, so this code is not complete yet. The `swapcontext` function would:
- Save the current context
- Restore the previous context

Now,  how do we implement a mutex?
It is easy, since interrupts do not happen and we should only deal with threads:
```C
void mutex_lock(mutex_t *m) { 
	if (m->locked) { 
		enqueue(m->queue, CurrentThread); 
		thread_switch(); 
	} else 
		m->locked = 1; 
} 

void mutex_unlock(mutex_t *m) { 
	if (queue_empty(m->queue)) 
		m->locked = 0; 
	else 
		enqueue(runqueue, dequeue(m->queue)); 
}
```
This is all very simple, but again, note that is is only atomic on a single CPU and with no interrupts!