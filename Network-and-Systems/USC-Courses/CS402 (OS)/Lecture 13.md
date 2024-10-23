# Mutex Implementation (Cont.)

## Multi-Processor, no Interrupt

We now consider straight threads on multiple processors.
The difficulty here lies in locking. Take this line of code:
```C
if (!m->locked) {
	m->locked = 1;
}
```
If two threads, execute this code in different CPUs at the same time, then they will both think that they have the lock!

![[Pasted image 20240401215923.png|400]]

This cannot be handled just with software, the hardware needs to support us.

### Compare-and-Swap (`CAS`)

We introduce the CAS operation, mostly implemented as a machine instruction. It can be described with the following C pseudo-code:
```C
int CAS(int *ptr, int old, int new) { 
	int tmp = *ptr; // get the value of mutex 
	if (tmp == old) // if it equals to old 
		*ptr = new; // set it to new 
	return tmp; // return old 
}
```
The processor guarantees that this is atomic, even with multiple processors, and to do this, the CPU implements a `LOCK` signal, that when set, prevents DMA from other cores. The signal is managed internally by the CPU and used seldom for some instruction (like CAS), so there is no danger of deadlocks.

Now, how would locking work? If the Mutex is unlocked (its internal value is 0), then we can lock it with `cas(&lock, 0, 1)`, which gives us this:

![[Pasted image 20240401220430.png|400]]

So when the CPU sees that a core has called `CAS`, it sets the `LOCK` signal and then DMAs the value at address `&lock` and then sets it if needed.

With this, we can now implement spin locks on multiple CPUs:
```C
void spin_lock(int *mutex) { 
	while(CAS(mutex, 0, 1)) ; 
} 

void spin_unlock(int *mutex) { 
	*mutex = 0; 
}
```
This spin lock isn't good, since if the mutex is locked, it will repeatedly call `CAS` and prevent other threads from inspecting the lock value, which would also mean preventing them from unlocking it!
So we need to minimize the amount of time we use `CAS`, and we can do it like this:
```C
void better_spin_lock(int *mutex) {
	while (1) {
		if (*mutex == 0) {
			if (!CAS(mutex, 0, 1))
				break;
		}
	}
}
```
This works, but is very wasteful, we should do better.

## Blocking Locks

On a uniprocessor, this implementation works:

```C
void blocking_lock(mutex_t *m) { 
	if (m->holder != 0) { 
		enqueue(m->wait_queue, CurrentThread); 
		thread_switch(); 
	}
	else 
		m->holder = CurrentThread; 
} 

void blocking_unlock(mutex_t *m) { 
	if (queue_empty(m->wait_queue)) 
		m->holder = 0; 
	else { 
		m->holder = dequeue(m->wait_queue); 
		enqueue(RunQueue, m->holder); 
	} 
}
```
With no interrupts and a single processor, it is easy to see why this is right, but why is it *wrong* with multiple processors? There is at least two race conditions on a multi-processor with this code:
1. If two cores execute `blocking_lock` at the same time, they will both get the lock (we have seen this before)
2. A more subtle race is the following. If (a thread in) core 2 has the lock and (a thread in) core 1 is trying to lock the mutex at the same time that core 2 is trying to unlock the mutex, we will have a problem.
   In particular, core 1 will see the lock is not available will attempt to go to sleep. Before it can do that, core 2 opens the lock and notices that the wait queue is empty, so it just sets `holder` and returns, core 1 will now go to sleep and never wakeup!

So how to solve this?
We can use spin locks here! Here is the first attempt:
```C
void blocking_lock(mutex_t *m) { 
	spin_lock(m->spinlock);
	/* ... rest of code ... */
	spin_unlock(m->spinlock);
} 

void blocking_unlock(mutex_t *m) { 
	spin_lock(m->spinlock);
	/* ... rest of code ... */
	spin_unlock(m->spinlock);
}
```
Right?
No! It will not work. If Core 1 attempts to lock an available lock, then it will block the spin lock, and attempt to go to sleep, however if before `thread_switch` finishes, core 2 tries to unlock the mutex, then it will be blocked by the `spin_lock` and will never return (note that when we call `thread_switch` we don't get to execute `spin_unlock` in the first call!)

Another attempt, with just the `lock` call:
```C
void blocking_lock(mutex_t *m) { 
	spin_lock(m->spinlock);
	if (m->holder != 0) { 
		enqueue(m->wait_queue, CurrentThread); 
		spin_unlock(m->spinlock);
		thread_switch(); 
	}
	else 
		m->holder = CurrentThread; 
	spin_unlock(m->spinlock);
} 
```
This has another problem though, there is a race condition. Assume core 2 has the lock and core 1 tried to get the lock and is about to go to sleep. Before the spin lock is opened, core 2 tries to unlock the mutex but gets blocked.

The moment core 1 puts itself on the wait queue and unlocks the spin lock, core 2 blasts away, and unlocks the mutex and dequeues the thread on the wait queue (which is thread 1 itself!) and puts it on the run queue. This schedules the dequeued thread to be run on **core 2**.
Now, core 1 calls `thread_switch`, it sees that it is on the run queue, and also starts running.

***The thread that used to be on core 1 is now running on both cores!!!***

The only way to solve this is by using spin locks in `thread_switch`, and that is not easy! This way of locking really does not work well.

## FUTEX

For $1 \times 1$ kernels, FUTEX (Fast User-level Mutex) is a saving grace, essentially:
- If the lock is open (most of the time), it gives you the lock right away with no Syscall
- If it is not open, it uses a Syscall to put you to sleep

The FUTEX gives you a struct that has a value and a queue of sleeping threads and two system calls:
```C
futex_wait(futex_t *futex, int val) { 
	if (futex->val == val) 
		// sleep on the futex queue inside kernel 
} 

futex_wake(futex_t *futex) { 
	// wake up one thread from wait queue if 
	// there is any ... 
}
```
The value of 0 means unlocked, else it is locked. It uses a combination of `CAS` and some other x86 features to implement atomic increase and decrease on the value.

>[!IMPORTANT]
>From now on, we live in $1 \times 1$ kernel world (which includes sixth edition Unix) and FUTEX is our basic lock. The FUTEX lock provides reliable and fast locking on a single or multiple processor that **receives no interrupts**.

# Mutex With Interrupts

The main (and really only nice) solution to handing interrupts while locking is to disable interrupts for a brief moment. There is however, still one problem.

Kernels can be preemptive or non-preemptive. A non-preemptive kernel will return to the same thread when it returns from the interrupt context, which means that disabling interrupts would be enough to shield the FUTEX lock.

However, if the kernel is designed to be preemptive, disabling interrupts won't be enough, since we can be kicked out of the CPU in the middle of a critical section in FUTEX. So, how to handle this?

The solution is very anticlimactic, disable both interrupts AND preemption!
Keeping track of whether or not preemption is enabled, can be done with a global variable. If preemption is disabled, then the kernel will not attempt to kick the current thread of the CPU. To protect the global variable from multiple threads on multiple CPUs, we just use a spin lock.

So in order to protect the threads from interrupts, we just need to disable preemption, then disable interrupts, do the lock/unlock code, and enable in reverse order.

**The order matters here! (First preemption, then interrupts, we'll see why shortly)**

## Sharing Variables With Interrupt Handlers

There are many kernel variables that can potentially be shared with interrupt handlers and kernel threads. This is similar to how a signal handler can share some variable with a kernel thread. We saw that this can lead to really nasty interactions (remember the thread that deadlocked with itself??)

So by similar reasoning, we cannot have code like this:
```C
int x = ...
void function_for_thread() {          void function_for_interrupt_handler() {
	++x;                                  ++x;
}                                     }
```
This can really evaluate to anything.

Again, the simplest solution would be to disable interrupts (or at least the one associated with invoking `function_for_interrupt_handler`) in the body of `function_for_thread`. This works well for non-preemptive kernels, but there is still a catch.

First, why do we even consider this? When will it even be reasonable to share things between interrupt handlers and kernel threads?
Well, for example, take disk IO actions!
```C
int disk_write() {
	start_IO();
	...
	enqueue(disk_wait_queue, currentThread);
	thread_switch();
	...
}

int disk_intr() {
	thread_t *thread; 
	... 
	// handle disk interrupt 
	...
	thread = dequeue(disk_waitq); 
	if (thread != 0) { 
		enqueue(RunQueue, thread); 
		// wakeup waiting thread 
	} 
	...
}
```
Note that disk IO completion comes with an interrupt, not from another thread!
So, for this, we propose that you mask disk interrupts when you use `disk_write`:
```C
int disk_write() {
	int oldIPL = setIPL(diskIPL);
	// rest of disk_write
	setIPL(oldIPL);
}
```
However, this does not really work. If the thread goes to sleep,  then disk interrupt remains masked and when it is actually delivered, it will remain pending forever! The sleeping thread would freeze, until some other code resets the interrupt mask.

You can't fix this by moving `setIPL(oldIPL)` to right before `thread_switch`, this would again have a race condition (for example, if you start the IO, set the IPL, if the IO finishes right away, the interrupt gets delivered to no one, and you go on the wait queue for an interrupt that was already delivered and sleep forever).

As we saw with blocking locks, something needs to be done **inside** the `thread_switch` function to handle this. And here, we can actually do that.

First, note that the run queue is accessed by all interrupt handlers potentially, and thus `thread_switch` also needs interrupt protection:
```C
void thread_switch() {
	int oldIPL = setIPL(HIGH_IPL);
	...
	
	...
	setIPL(oldIPL);
}
```
But the main modification that we need to do here is the following:
- If the run queue is not empty, then nothing needs to be done
- If the run queue is empty, then it is possible that a thread is blocked because it was waiting for an interrupt, so we should enable ALL interrupts. This can potentially cause us to get yanked by the CPU to handle the interrupt, but that is ok, since nothing is happening with the run queue right now.
  As a result of the interrupt finishing, a thread might be on the run queue now, so we should disable all interrupts again, and recheck the run queue.
  We do this until some work comes in for us.

So right after disabling interrupts, we will do:
```C
while (queue_empty(run_queue)) {
	setIPL(0);
	setIPL(HIGH_IPL);
}
```
This is naïve though, since it is busy waiting. The correct solution is to block. Since nothing is on the run queue, we just have to wait completely till an interrupt knocks on the door, and so we can just halt the CPU:
```C
while (queue_empty(run_queue)) {
	setIPL(0);
	HLT               // Machine instruction to halt the CPU
	                  // We will not discuss the multi-processor case of this
	setIPL(HIGH_IPL);
}
```
But this has a problem!
If you enable the interrupt, it can get delivered right away, in that case you could potentially sleep forever if you halt without checking. So we will do:
```C
while (queue_empty(run_queue)) {
	setIPL(0);
	if (!queue_empty(run_queue)) {
		setIPL(HIGH_IPL);
		break;
	}
	HLT               // Machine instruction to halt the CPU
	                  // We will not discuss the multi-processor case of this
	setIPL(HIGH_IPL);
}
```
Note that `HLT` itself atomically enables interrupts and goes to sleep, so if interrupts get delivered when `HLT` is executing, it aborts.

## With Preemption

There is one last thing to discuss. 
With preemptive kernels, we mentioned that we should FIRST disable preemption and THEN interrupts.

The reason for this is very subtle, but basically, if we do it the other way around, assume the following:
1. The current interrupt mask value is 5, we disable interrupts by setting it (the mask) to 32 and then keeping the 5 value somewhere in the original thread.
2. We get preempted, we get into a thread that change the current mask (for whatever reason) to say, 32. It too will keep the previous value (32!) somewhere before it goes on with doing the rest of its code.
3. The thread however gets preempted again, and comes back to us (unlikely, but just assume it happens).
4. We disable preemption, do everything in peace, then enable it again, but lucky for us, this time the kernel does not kick us and we return the interrupt mask value to what it was, **32**.
5. The kernel resumes execution of the other thread, which finishes its work and return the interrupt mask to **32**!!!

So interrupt values can potentially get messed up, that is why you first disable preemption and then interrupts!