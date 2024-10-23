# Synchronization (Cont.)

## Guarded Commands (Cont.)

Last lecture, we gave a general recipe for converting guarded commands to POSIX.
Let's do another example:

### Example: Readers-Writers 

Multiple reader and writer threads are accessing some bundle of shared data. We want to synchronize this process. With guarded commands we can write:

```c
reader() {
	when (writers == 0) {
		readers++;
	}
	/* read */
	[readers--;]
}

writer() {
	when ((writers == 0) && (readers == 0)) {
		writers++;
	}
	/* write */
	[writers--;]
}
```
We don't check for `readers == 0` in `reader()` since we WANT to let multiple readers in when there are no writers, because why not?

This code is actually quite tricky, make sure you understand why it can let multiple readers work, but only one writer at a time.

That said, let's note the guarded command in `readers()`. If you want to implement it to the letter, we'll get:
```c
pthread_mutex_lock(&mutex);
readers--;;
pthread_cond_broadcast(&cv);
pthread_mutex_unlock(&mutex);
```
However:
- Why wake everyone up? The readers will do what they want, however only a single writer will be admitted anyway, why just not wake a single one up?
- Why even wake anyone up always? If the number of readers is larger than 1, the awoken writer will just go back to sleep anyway!

This shows you an important thing about our previous implementation of guarded commands. While they are **correct**, they **may not be efficient!**. 
To optimize the above code, one would do:
- Replace `pthread_cond_broadcast` with `pthread_cond_signal` to wake up just one thread.
- Check if `readers == 1` before even signaling in the critical section.

That said, there is a little problem here, *there is no way to say whether or not the wake up thread is a reader or a writer!* If you care about that, use two condition variables instead of 1.
Doing this, you would then:
- Broadcast all readers when necessary using it's CV
- Signal a single writer when necessary using it's CV

So, putting all of this together, the POSIX implementations will be:
```c
reader() {
	pthread_mutex_lock(&mutex);
	while (!((writers == 0))) {
		pthread_cond_wait(&reader_cv, &mutex)
	}
	readers++;
	pthread_mutex_unlock(&mutex);
	// READ
	pthread_mutex_lock(&mutex);
	if (--readers == 0)
		pthread_cond_signal(&writer_cv, &mutex);
	pthread_mutex_unlock(&mutex);
}

writer() {
	pthread_mutex_lock(&mutex);
	while (!((writers == 0) && (readers == 0))) {
		pthread_cond_wait(&writer_cv, &mutex);
	}
	writers++;
	pthread_mutex_unlock(&mutex);
	// WRITE
	pthread_mutex_lock(&mutex);
	writers--;
	pthread_cond_signal(&writer_cv);
	pthread_cond_broadcast(&reader_cv);
	pthread_mutex_unlock(&mutex);
}
```
This is nice and efficient.
So we need a new problem to force us to rewrite it!

### Starvation And Live-lock

The above code *starves* the writers, meaning that in certain cases, it will always give higher priority to readers instead of writers.

To see this, assume that new readers will just keep coming, so will new writers, who do you think will have a higher chance to access the data? Well of course it will be readers! Why?

If a reader starts reading the data at some point, then `writers == 0`. Thus as long as readers start coming in, the number of readers can increase arbitrarily, while keeping `writers == 0` true. Thus, readers can keep access to the resource for an arbitrary amount of time, even if many many writers are waiting!

On the other hand, if some writer finishes it's job, it means that `writers == 0` and `readers == 0`, but now, all bets are off! Both critical sections are now open, only luck decides which one gets executed first, and if it's the readers, the writers will be in big trouble!

So even if the wake up call is distributed fairly across all sleeping readers and writers, the writers will always be at a disadvantage! This is what we call **Starvation**, also called a **Live-lock** in some contexts since it is virtually similar to a deadlock, and for a kernel implementation, it can be particularly ruinous in terms of efficiency.

Before we start pontificating about general solutions to this, we can just solve the previous example again. 

We start by examining the root cause of the problem, as we said, at some point both critical sections can be opened, and that is what leads us into the starvation of the writers, however, that is just the nature of this problem! It is a logical implication of the guards that cause this, and we can't touch that without changing the problem fundamentally.
So the only course of action we have, is to "shut the door on the greedy", which means to artificially give the writers the power to stop readers from coming in.

How will you do that? Well, one solution might be to distinguish between writers that are *waiting* and writers that are *writing* (i.e. modifying the data). To this end, we introduce the variable `active_writers` to indicate the number of writers that are writing to the data. Only when this is 0, can we start writing to the data for safety. 
As for reading, to give the "artificial advantage" to the writers, we simply just say that readers can come in, if and only if **all writers are gone** (i.e. this conditions on `writers` like before, instead of `active_writers`).

So the `reader()` does not change, but for writers:
```c
writer() {
	[writers++;]
	when ((readers == 0) && (active_writers == 0)) [
		active_writers++;
	]
	[
		writers--;
		active_writers--;
	]
}
```
And this can be converted to POSIX like before.
Finished? Well not really, one can still say that "now you are being unfair to readers!", and that is correct, but at least writers won't be starved to death!

## Barrier Synchronization

A "barrier" is some point in the execution of a thread, where it is obliged to wait for all other sibling threads to "catch-up" to it exactly at that point, before it is allowed to progress further. Once all threads reach that point, the barrier will collapse and execution will resume like before.

This form of synchronization is very much needed in parallel computing (mostly in aggregation stages mid execution), so we need to talk about this.

>[!EXAMPLE] Naïve Solution
>Put all the barriers in the main thread, and join with all children threads before it. Once we return to main, create new threads post-barrier and resume execution, easy!

This is correct, however, **join and create directives are very expensive!**, there are a lot of setups and logistics to go through, things are much, MUCH worse on multiple CPUs as well, which is exactly where barrier synchronization is needed the most!

The most widely accepted solution to this is the aptly named **Thread Pool Abstraction**. The thread pool is just a bunch of threads that are allocated exactly once, and never die unless exactly said that they should.

Once we cross a barrier, we borrow threads from the thread pool instead of creating them, and the thread pool will handle joining them internally with it's own API. This is quite different from what we did and a fair amount of details go into making this work, so let's push this to the side for now.

Let's do a makeshift solution for this.
We count the number of threads in the barrier with some global variable, and a CV for queuing up sleeping threads at the barrier. So we can do:
```c
int count = 0;
pthread_mutex_t m;
pthread_cond_t BarrierQueue;
void barrier_sync() {
	pthread_mutex_lock(&m);
	if (++count < n)
		pthread_cond_wait(&BarrierQueue, &m);
	else {
		count = 0;
		pthread_cond_broadcast(&BarrierQueue);
	}
	pthread_mutex_unlock(&m);
}
```
Voila, we are done!
Of course not! As if things could be easy. 
It turns out `pthread_cond_wait` can return **spontaneously**! What this means, is that when you call it, it's possible that you might return instantly!

Read this if you want to see what's going on.

>[!FAQ]- Is POSIX's `wait` Bugged?
>This really looks like a bug, why should `pthread_cond_wait` return instantly?!
>This behavior is called "spurious wakeup", and is a well documented POSIX "quirk" of the library, it is deliberately ignored however, for two reasons:
> - **As long as you guard a wait with a loop**, this behavior will correct itself.
> - Solving it makes the implementation sluggish for ALL executions.
>   
> To see why this happens would require us to take a good look at the POSIX implementation and that is pretty advanced C code that we are not ready for yet. You can look [here](https://pubs.opengroup.org/onlinepubs/009604599/functions/pthread_cond_signal.html) for some deeper explanation, but here is some summary quoted from it:
> >On a multi-processor, it may be impossible for an implementation of _pthread_cond_signal_() to avoid the unblocking of more than one thread blocked on a condition variable. ... 
> >
> >The effect is that more than one thread can return from its call to [_pthread_cond_wait_()](https://pubs.opengroup.org/onlinepubs/009604599/functions/pthread_cond_wait.html) or [_pthread_cond_timedwait_()](https://pubs.opengroup.org/onlinepubs/009604599/functions/pthread_cond_timedwait.html) as a result of one call to _pthread_cond_signal_(). This effect is called "spurious wakeup". Note that the situation is self-correcting in that the number of threads that are so awakened is finite; for example, the next thread to call [_pthread_cond_wait_()](https://pubs.opengroup.org/onlinepubs/009604599/functions/pthread_cond_wait.html) after the sequence of events above blocks.
> >
> >While this problem could be resolved, the loss of efficiency for a fringe condition that occurs only rarely is unacceptable, especially given that one has to check the predicate associated with a condition variable anyway. Correcting this problem would unnecessarily reduce the degree of concurrency in this basic building block for all higher-level synchronization operations.
> 

OK, so we can guard the wait with `while(count < n)` to handle the spurious wakeup, are we done now??

Well, it's even **worse** now!
When a thread crosses the barrier, it sets `count` to 0, but this means that once the other threads wake up, they see that `count < n` is still true and go back to sleep! All `n-1` threads blocked on the barrier will sleep for all eternity!

In general, **guarding the condition wait with the number of threads in the barrier is always problematic**, you need a separate variable for the guard.

In POSIX jargon, barriers are identified with a *generation number* starting from 0. Threads at first all belong to generation 0, and once a barrier collapses, they move on to the next generation. Barrier waits are guarded with the generation number of that barrier, and once the barrier collapses, the first thread to cross will set `count` to zero and increment the generation.

So we'll end up with the following in each thread:
```c
if (++count < n) {
	int my_generation = barrier_generation;
	while (my_generation == barrier_generation)
		pthread_cond_wait(&BarrierQueue, &m);
}
else {
	count = 0;
	barrier_generation++;
	pthread_cond_broadcast(&BarrierQueue);
}
```

# Thread Safety

The term "thread safety" was first created during the advent of multi-threaded programming, which happened much after Unix's first release, and was used to identify problems that happened when previous Unix code was used in multi-threaded programming.

These issues mostly arise from global variables (say `errno`) and shared data (i.e. `printf()` formats). 

These days, making codes thread-safe is a MUST! 
You NEED to keep this in mind when you create a library that is supposed to be used by other people.

>[!FAQ] Reentrant vs. Thread-Safe
>Reentrant code is sometimes confused with thread-safe code. Let's be clear:
>- **Reentrant Code** is code that can be interrupted and executed again, without the previous execution finishing.
>- **Thread-Safe Code** is code that can be performed from multiple threads safely. Even with shared variables.
> 
>These definitions are orthogonal, they describe different properties, but are commonly confused. 

We focus on thread-safe in this class.

## Example: `errno`

The `errno` is a global variable that in UNIX is used to see what went wrong during a function call. Inside that function, the function will set `errno` to some specific value based on what happened. So, in a threaded implementation, if two calls to the same function go wrong for *different reasons*, one of them will overwrite the `errno` set by the other! This code is not thread-safe.

So how would you solve this? (remember, this extends to all other global variables in UNIX). Well, you can create a global variable specific for *that thread*. Only global variables used for thread synchronization need to be shared among them, `errno` is NOT one of them, those variables need to be user defined.

Therefore, we can replicate these variables across all threads. These are stored in what we call the **Thread Specific Data** block in the TCB.

In UNIX, the engineers came up with some really elegant solution for `errno`, namely, they just added the following code to the UNIX headers:
```c
#define errno __errno(thread_id)
```
And recompiled the code!

This replaces each instance of `errno` within each thread with the thread-specific version of it that is mapped to their thread identifier, all we need is to implement `__errno` and we are done!, in fact, we don't even need to do that! we just leave it to the POSIX folk.

## Example: `gethostbyname()`

In UNIX, a function exists called `struct hostnt *gethostbyname(const char *name)` which gets a hostname and returns it's namespace (i.e. all IP addresses for it from the DNS).

You might sense that this definition is confusing …
It is returning a pointer but to *who*? It definitely isn't in the arguments, is it?

Well, it's actually a pointer to a *global* variable! (a really questionable decision, but ok). So it's not hard to see that this code isn't thread-safe.

So, let's do it like we did with `errno`, right?
Well no, we NEED to use `malloc` here, but that requires separate directives to give the thread the pointers to those heap variables, this is not as easy as `errno`.

So UNIX engineers just throw up their hands and said that we just implement the *reentrant* version of this function, and leave the allocation of buffers and variables to the user!

So in UNIX, the thread safe version of this function is:
```c
int gethostbyname_r (
	const char* name,
	struct hostnt *ret,
	char *buf,
	size_t buf_len,
	struct hostnt **result,
	int *h_errnop
)
```
Which is much more complicated and demanding from the user. But it is preferred, since:
- The programmer NEEDS to know what they are doing (which is a desirable state for a multi-threaded programmer)
- The programmer NEEDS to keep the pointer to all buffers in the caller, a good practice in general.

## Example: `printf()`

Using `printf()` on shared variables is not thread-safe.
Solution?
Just lock a mutex before using `printf`! This isn't `printf()`'s fault really, so we don't change it, it is just not supposed to be used with shared variables (i.e. `printf()` is reentrant, but it is not thread-safe).

# Deviations

**Thread Deviation** means to ask a thread to stop what it's doing, do something else, and return as if nothing had interrupted it in the first place.

The observant among you might instantly recognize that this is just context-switching, but in a thread level. In fact, it is *exactly* that, since as we said before, threads are just abstractions of the CPU.

So instead of saving the PCB before switching, we save the TCB, done.

However, there is another example of deviation that is most interesting to us, and that is *cancelation*, asking a thread to stop and terminate cleanly (by clean, we mean we want it to deviate to a code sequence that cleans up everything it did or rolls-back something it did, and then calls `pthread_exit`).

The main mechanism for the implementation of all of these, is Unix's **Signaling** directives.

## Signals

Signals are a way for us to talk to threads/processes.

The original purpose of signals in pre-threading Unix was to:
- Tell a process to stop without calling `exit` directly
- Report really bad instructions (like divide by zero, or `SEGFAULT`) to users (note that recognizing the `SEGFAULT` needs a trap, but actually reporting it requires an upcall, and that upcall is implemented with signals)

A typical signal is the *keyboard-input kill signal*, which in Unix is *ctrl + C*, or `^C`.

>[!BUG] General Misconception
>Signals are not Software Interrupts. They are called *callback mechanisms* and are implemented using upcalls.

Signals have the same abstractions used for hardware interrupts:
- Signals can be blocked, similar to how interrupts can be disabled.
	- Blocked interrupts will become "pending", just like blocked hardware interrupts.
- If Signal is not blocked, it will be delivered as soon as possible, but not instantly (just like hardware interrupts).

Signals however, since they are software, have some other nice functionalities. It is very important for the OS to recognize who sent a signal and why they did it, and what it needs to do. More importantly, unlike interrupts that can be blocked indefinitely, some signals are really strong and bypass all directives, even if the user or the OS said they shouldn't.

A good example (and a really useful one) is *Force Kill Signal* or `SIGKILL`, which unlike the keyboard interrupt `SIGINT`, bypasses everything and completely wipes out the program and removes it's core dump without even needing a handler.

In Unix, the shell can deliver signals to a process given it's PID using the `kill` Syscall. To use this, one would do `kill -sig PID`, where `sig` is the signal number that you want to send. These are Unix specified, so for example, `SIGINT` is 2 and `SIGKILL` is 9. Thus if a process with PID `123` is running:

- `kill -2 123`: Interrupt process 123
- `kill -9 123`: Kill process 123 without caring what it's doing

There is a similar mechanism in POSIX with `pthread_kill`, ***DON'T USE IT!***
