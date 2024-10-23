# Signals

## Handling Signals

There are two ways to handle signals:
- **Asynchronously:** With *signal handlers*, delivery happens at random, since the system catching the signal runs in parallel with the CPU.
- **Synchronously:** Using `sigwait` in a thread specifically designed to catch signals

### Signal Handler

Each process may only have at most one single handler for a signal, which is treated as part of that process's context.

To specify these, we use `signal/sigset` in `signal.h`, which are essentially the same function. These will return the function that shall be run, when a signal is received (thus, a default value is returned for each signal if needed).

```c
#include <signal.h>

// Defines function pointer *sighandler_t to be pointer to a function that accepts
// an int (signo) and returns nothing.
typedef void (*sighandler_t) (int);

// signo: Which signal to catch
// handler: The actual function to call after catching
sighandler_t sigset/signal(int signo, sighandler_t handler);
```
**Note:** Inside the signal handler of a particular signal, that signal itself is BLOCKED! (for obvious reasons).

>[!FAQ] A Side-Note
>In some systems, when a handler finishes, the system will switch to the default handler for the next execution (identified with `SIG_DFL`). 
>In this case, you MUST set the signal in the function again!
>There is also `SIG_IGN` which just ignores the signal (i.e. if `^C` is detected, just ignore it).

>[!IMPORTANT]
>We'll see what this means later, but signal handlers are part of the **execution context** of a process. This means that you can save handlers, switch to a new one, and then restore the old handler in the same process.

To see this in action, here is a stupid program:
```c
#include <signal.h>
int main(void) {
	void handler(int);
	sigset(SIGINT, handler);
	while(1)
		;
	return 1;
} 

void handler(int signo) {
	printf("I wasn't killed! HA!\nSIGNO = %d\n", signo);
}
```
Upon running, this program never terminates, even if you use `^C`, all it does is laugh at you and print the signal number of 2. You need to kill this with `kill -15 <pid>`. The default action of 15 is to kill programs no questions asked.

If the side note above holds water though (which in Linux mostly does), the signal handler will return to default, and if you use another `^C` after the first one, it may very well kill your program.

Asynchronous signal catching can be prone to race conditions when multiple signals are present. This can be handled asynchronously, but it's just better to switch to synchronous handling in this class. To fix the asynchronous case, we need to make the code *asynchronous-signal safe*, which just means that no data structure in the code gets corrupted by asynchronous signals.

#### Example 1

Here is an example of a code that isn't safe for asynchronous handling:
```c
sigset(SIGALRM, doSomethingInteresting);
struct timeval waitperiod = {0, 1000}; // interval of 1ms
struct timeval interval = {0, 0};
struct itimeval timerval;

timerval.it_value = waitperiod;
timerval.it_interval = interval;

setitimer(ITIMER_REAL, &timerval, 0); // Send SIGALRM in 1ms
pause(); // wait for any signal
```

Turns out, this code has a race condition. It is supposed to set a timer for 1ms, go to sleep by calling `pause` and wake up after `SIGALRM` is delivered. Basically, `pause` sets up a generic signal catcher for everything, and then goes to sleep.

BUT, imagine that between the last two lines (i.e. before `pause`), the CPU is taken away from you. Remember that the timer is *already set*, thus, if you don't get the CPU before 1ms, the timer will go off. 
Assume this happens, so when you come back, you'll see that `SIGALRM` is pending, therefore you deviate and execute `doSomethingInteresting`, and comeback to where you were, which was right before calling `pause`. Thus, you proceed to call `pause`, and sleep forever!

This is a race condition, but the result isn't really a deadlock per se, but it might as well be! The only way to wakeup this program is to send a signal, but that may not be possible based on what program you wrote.

#### Example 2

Here is a real example …
In the olden days, GUIs weren't widespread, and those that did exist, were a pain to write/use. So, one way to actually show the user the state of running processes, was to:
- Do it periodically
- Do it when the user signals you (like with `^C`)

Let's consider the latter, imagine that we have a really long process, and we want to check on it's status by calling a signal handler set for `^C`.
```c
#include <signal.h>
computation_state_t state;
int main(void) {
	void handler(int);
	sigset(SIGINT, handler);
	long_running_process();  // Some long process that updates
							 // the 'state' variable.
	return 0;
}
void handler(int signo) {
	display(&state);
}
```
Every time the handler is called, it will display the process state with `display` (like it might say something like `The process is 90% complete`). 
This code works nice most of the time, but let's assume the following scenario.
- The code is running normally, and some procedure inside the `long_running_process` is accessing `state`. For example, we might be linking or unlinking a linked list in the process.
- `SIGINT` is delivered, and the code deviates, however, note that this might happen **while we were processing the linked list in `state`**.
- The `display` function now might try to access that linked list (which might have broken `next` or `previous` pointers right now), it may either run into a segmentation fault, or go into an infinite loop, or just spit out garbage!

Can this be solved with a mutex? You would be forgiven for thinking that this would solve this, however, this can have a really interesting outcome.

We have seen how to handle critical section code, doing that now, we would have to lock the mutex before we call `display` or before we access `state` in the long running process, and unlock it once we are done. 

However, that won't work either. Remember that when we handle signals, the OS borrows the running thread and makes an upcall with it, this time with the signal handler function instead of the code that was running previously (this is how deviation is done with all threads). Therefore, you cannot return to your original code, without finishing the handler.

So how would that work here? Well, assume we are in the exact same scenario as before, we are in the middle of updating `state`, and therefore, we should have the mutex locked. Now, the user presses `^C` and the signal is delivered by the OS. The code now deviates to `handler`, which also has a critical section code.  The code will now try to lock the mutex again!, but this means that the code will now go to sleep forever, since the mutex was already locked!

As hilarious as it is, the code will **deadlock with itself!** (and yes, to make this more embarrassing, this is happening with just a single thread!).

>[!FAQ]
>Can't you just not lock the mutex in `display`? The mutex is already locked when you access it anyway, so why bother?
>
>Well, that is just the previous example with extra steps! 
>We deviated in the middle of an operation on `state` and therefore it might have a broken linked list. If we remove the mutex locking in `display` it can still SEGFAULT or go into an infinite loop! There is no way to save this code.

The main thing that is causing the problems is that signal handlers just have too much priority, they can even bypass mutex locks. The thing is, *we already have seen this problem*.
This is technically the same as interrupts in the middle of `pthread_mutex_lock`, which can also result in a deadlock just by calling the mutex. How did we solve it?

Well, the POSIX folks have different solutions, but a simple one might be to just *disable interrupts*, why not do that here? Just disable signals when running critical section code in the long running process!

And that, is the correct solution as a matter of fact. Disabling interrupts might be a bit iffy, but disabling signals is an accepted practice and one that has very good and legitimate reasons, and that is what you need to do when doing these asynchronous signal handlings. I hope you can now see how this might become complicated in a large and nested code.

#### Masking/Blocking Signals

To block a signal is called *masking* in OS jargon. You can mask a signal by marking it in the thread control block with a single bit. In fact, that is the most widely used solution, since there is exactly 32 signals, and that can be nicely represented by 32 bits.

Thus, to block a specific signal, just set it's corresponding bit in the TCB to be 1 (e.g. to block `SIGINT`, set the 3rd bit to 1 and the rest as 0).

>[!IMPORTANT]
>Threads inherit their signal masks from their parent threads by default.

To examine or change signal mask in the calling process/thread, we can use the following:
```c
#include <signal.h>
int sigprocmask(
	int how,
	const sigset_t *set,
	sigset_t *old
);
```
Here:
- `set` is the 32 bit value we mentioned
- `how` is one of the following:
	- `SIG_BLOCK`: The actual mask is the OR of it's current value and `set`
	- `SIG_UNBLOCK`: Same as above, with AND
	- `SIG_SETMASK`: Just copy the given value to the mask
- `old` will contain the previous value of the mask. This is useful for restoring context.

To manipulate the 32 bit value, you can:
- Use `sigemptyset(sigset_t *set)` to unblock all.
- Use `sigfillset(sigset_t *set)` to block all.
- Use `sigaddset(sigset_t *set, int signo)` to add a specific signal number (like 2 for `SIGINT`) to the set, blocking it.
- Use `sigdelset(sigset_t *set, int signo)` to remove a specific signal number from the set, unblocking it.

#### Fixing The Examples

To fix example 1, we will:
```c
sigset_t set, oldset;
sigemptyset(&set);
// Set to block SIGALRM
sigaddset(&set, SIGALRM);
// Block SIGALRM and save the previous context
sigprocmask(SIG_BLOCK, &set, &old_set);
// Set timer for 1ms
setitimer(ITIMER_REAL, &timerval, 0);
// Set to block all except SIGALRM
sigfillset(&set);
sigdelset(&set, SIGALRM);
// Wait for the signal safely (see below)
sigsuspend(&set);
/* In this line, SIGALRM is masked again */
// ... do things ...
// Restore the context
sigprocmask(SIG_SETMASK, &oldset, (sigset_t *)0);
```
Here, `sigsuspend()` atomically unblocks the signal and waits for the signal.

![[Pasted image 20230226020535.png]]

>[!IMPORTANT]
>Note that before calling `sigsuspend` on `SIGALRM`, we had already *blocked* it! This needs to be done every time you use this function.

For example 2, you also need to do the same, just block signals in the critical sections and restore the context afterward. Since the handler can only happen when a signal is delivered (and therefore, only when a signal is unblocked), you don't even need a mutex at all! 

```c
#include <signal.h>
sigset_t set;
computation_state_t state;
int main(void) {
	void handler(int);
	sigemptyset(&set);
	sigaddset(&set, SIGINT);
	sigset(SIGINT, handler);
	long_running_proc();
	return 0;
}

void long_running_proc() {
	while (long_time) {
		sigset_t oldset;
		// non critical stuff
		sigprocmask(SIG_BLOCK, &set, &oldset);
		update_state(&state);
		sigprocmask(SIG_SET, &oldset, 0);
		// non critical stuff
	}
}
```
### Summary

Let's split the scenarios:
- In one, you are waiting for an asynchronous event that **you** generated (like the timer example). 
  To implement this, you must:
  1. Block that event
  2. Do something that WILL generate that event some time in the future
  3. Unblock and wait on it **in an atomic operation**

- In another, you are waiting for an asynchronous event that **someone else** generated (i.e. waiting for a guard to become true in a guarded command).
  To implement this, you must:
  1. Block that event
  2. **Check if the event is generated or not**, if not, unblock and wait for it in an atomic operation.

## Signals In Threads

We've seen how to handle signals in a single process, but how about threads?

It's important to know that signals were implemented in UNIX much before threads were created, and thus, signals are delivered to **processes**, not threads. 
In POSIX, the implementation follows the following general rule:

>  When a signal is generated, the result will be handled by a randomly chosen thread that has not blocked that signals, or ignored. This may be the main thread, or any child thread.

In light of this, if you are catching a specific signal in a thread, the only way to guarantee that it will work correctly, is to **block the signal in any other thread**.

### Synchronous Signal Handling

In light of the above rule, one can come up with a different way of handling signals all together. Instead of creating handlers, which borrow a thread and run an upcall, *why not just create a completely separate thread to wait and handle the signal*?

And indeed, why not?
This is the synchronous way of handling signals, and does not even need handlers at all. All we need to do is:
- Protect critical section code with mutex like before
- Create a separate thread that will run code specifically for waiting and handling a signal
- Block that signal in all other threads
- Make sure that we unblock and wait for the signal in an atomic operation in the associated thread

Not only does this gel nicely with how mutex is used in POSIX, it just ignores all the other problems we have to solve with asynchronous signal handling. There is cons here, but we are going to ignore them.
**As far as this class is concerned, this is the ONLY way that you should handle signals.**

So, returning to the long process example, we can rewrite it:
```c
int main(void) {
	pthread_t thread;
	sigemptyset(&set);
	sigaddset(&set, SIGINT);
	sigprocmask(SIG_SET, &set, 0);
	// Now, this thread and all threads created by this thread, will
	// block SIGINT
	pthread_create(&thread, 0, monitor, 0);
	long_running_process();
	return 0;
}

void *monitor() {
	int sig;
	while (1) {
		sigwait(&set, &sig);
		pthread_mutex_lock(&m);
		display(&state);
		pthread_mutex_unlock(&m);
	}
	return 0;
}

void long_running_process() {
	while (long_time) {
		// non critical stuff
		sigprocmask(SIG_BLOCK, &set, &oldset);
		update_state(&state);
		sigprocmask(SIG_SET, &oldset, 0);
		// non critical stuff
	}
}
```
As you can see, the code for the actual running process does not even have to be changed, as far as it's concerned, `SIGINT` isn't even supposed to be handled!

You can see we use the function `sigwait`, and as you might have guessed, it atomically sets the thread mask to be `set` and waits for any signal in that set to happen, and when it does, it will put that signal number in `sig` and return.

>[!BUG] Common Error
>The `set` variable means different things here:
>- In general, when a signal is set to 1 in it, it means that it will be blocked.
>- In `sigwait`, when a signal is set to 1, it is actually technically unblocked! Since we want actually catch it.

>[!IMPORTANT] Small notes on `swigwait`
>- `sigwait` is a blocking function.
>- If handlers for any signal in the set given to `sigwait` previously existed somewhere, they will be  **ignored**, and instead only `sigwait` will return.
>- You should block the particular signals in other threads, if not, we have no idea who will actually catch the signal first.
>- Once again, `sigwait` is atomic, note this figure again:
>  
>![[Pasted image 20230226030130.png]]

## Signals During Syscalls

If a signal happens during a system call, 3 approaches may be chosen to deal with it:
- Deal with the signal *after* the Syscall is finished
- Interrupt the Syscall, deal with the signal and then resume the Syscall
- Stop the Syscall, deal with the signal and immediately return, reporting that something happened. 

Most systems prefer the 3rd approach. With this, the Syscall is left unfinished, and instead, `errno` will be set to `EINTR` (more precisely, it's the error number for the thread that was deviated to handle the signal).

>[!FAQ] Why Do This?
>The main concern here is the particular case where the initial thread is a kernel thread, but the signal handler attached to that signal is code executed in the user space.
>In this scenario, it's possible that the signal handler may attempt to call a Syscall in the user space, *before the Syscall in the kernel space was finished!!*. This can lead to an absolute mess ... since the Syscall executed by the user space may not even return correctly if the Syscall that the kernel was using was extremely critical.
>
>**Fun Fact:** Remember "spurious returns" in `pthread_cond_wait`? Well, turns out the above concern is one of the reasons that the developers decided to keep it that way! (let's not get bugged down explaining why though ...)

### Examples

To show how messy things can get when a thread is borrowed to deliver a signal, let's consider the good old `read` and `write` Syscalls.

Here is how you could code with `read` such that it will be robust to interrupts:
```c
while ((return_val = read(fd, buffer, buf_size)) == -1) {
	if (errrno == EINTR) {
		// We were interrupted, retry ...
		continue;
	}
	// Something went REALLY wrong ...
	perror("Big trouble!")
	exit(-1);
}

if (return_val == 0) {
	printf("End of input\n")
}
```
Since `read` never modifies the data, it is safe to retry it arbitrarily many times.
Not with `write` though!
```c
remaining_to_write = total_to_write;
bptr = buf;
while (1) {
	num_written = write(fd, bptr, remaining);
	if (num_written == -1) {
		if (errno == EINTR) {
			// Early interrupt!
			continue;
		}
		perror("Big trouble!")
		exit(-1);
	}
	if (num_written < remaining_to_write) {
		remaining_to_write -= num_written;
		bptr += num_written;
		continue;
	}
	// Finished ...
	break;
}
```
Do we really want to write code like this??
Well, most people say "absolutely not!". Thus, we make the following unwritten rule for pretty much anything in this class:

> [!IMPORTANT] General Rule For Signals And Threads
> All threads will block all signals!
> Signals may only be handled synchronously by using a thread which exists solely to catch signals.
> 
> Another important thing, is to remember that signal handlers are a pretty weird context. What this means, is that quite a significant portion of the code executed by a signal handler cannot be tracked or rolled back (e.g. if you set a breakpoint in a signal handler in GDB, you will get inconsistent results when using `where`).
> 
> It is paramount that **you do as little as possible in a signal handler and get out as soon as you can**.

### Thread Cancelation

We now come to one of the most important usages of signals in threaded programming. 

It is fairly common to have a "master" thread, a thread that somehow coordinates all child threads and decides how the computation must move forward, however, in the particular case that this master thread finishes or exits for whatever reason, unless we use `join`, we need to find a way to tell the child threads (or any other as a matter of fact) to stop.

In OS jargon, this is called **thread cancelation**, and while very useful, there are also many concerns that we should have about an implementation of it. For starters:
- You should be careful *when* you cancel a thread.
	- Canceling in the middle of unlocking a mutex or before it, may cause the lock to remain locked forever and create a deadlock.
	- If processes are left unfinished, all sort of inconsistent data structures and memory corruption bugs can happen.
- You should do a "clean up" on that thread, to prevent memory leaks or leaving mutex locks locked.

#### POSIX Cancelation

Just like signals, cancelation needs to be explicitly enabled/disabled. To do this, we use:
```c
int pthread_setcancelstate(
	{PTHREAD_CANCEL_ENABLED, PTHREAD_CANCEL_DISABLE},
	&old_state
)
```
Another important state factor is cancelation *type*:
```c
int pthread_setcanceltype(
	{PTHREAD_CANCEL_ASYNCHRONOUS, PTHREAD_CANCEL_DEFERRED},
	&old_type
)
```
Here:
- **Asynchronous Cancelation** means that after a cancelation call happens and becomes pending, then it will be delivered asynchronously, meaning that at any point that the thread notices that it has a pending cancelation call, it will immediately deviate from whatever it was doing and call `pthread_exit`.
- **Deferred Cancelation** means that cancelation is delivered, when the canceled thread reaches a **cancelation point**, a particular line that checks whether or not a cancel call is pending, and when it sees that there is such a call, it will call `pthread_exit`.

By default, cancelations are **enabled** and **deferred**, and you should have a very good reason to change that!

Cancelation call for a thread is requested with `pthread_cancel(thread)`.

General POSIX cancelation rules are:
- Cancelation calls are non-blocking.
- If cancelation in the destination thread is disabled, it will keep the request of `pthread_cancel` in pending, forever.
- If cancelation in the destination thread is enabled, it will either act on it immediately in asynchronous cancelation, or wait until it reaches a cancelation point if it is deferred.

##### Cancelation Points

Cancelation points are parts of the code where it is "safe" to deviate. These include (but are not limited to):
- File Syscalls (`read`, `write`, `close`, `open`)
- All forms of sleeping (`nanoslepp`, `sleep`, `pause`, etc.)
- Most "waits" (`wait`, `sigwait`, `waitpid`, `pthread_cond_wait`, `sigsuspend`, etc.)

The important things that it does **NOT** include are:
- `pthread_mutex_lock`
- `pthread_mutex_unlock`

There is the function `pthread_testcancel`, which when called, checks for any pending cancelation and if there are, immediately calls `pthread_exit`. This essentially sets a manual cancelation point. It is useful for codes that really may not have a valid cancelation point in a really long process.

##### Cancelation Cleanup

Immediately calling `pthread_exit` without cleanup is a really bad idea, therefore, POSIX allows us to specify a stack of cleanup handlers for any thread, which when we call `pthread_exit`, will be automatically executed by the library. These may include:
- Closing file pointers
- Freeing dynamic memories
- Closing sockets
- Unlocking a mutex
- Broadcasting on a condition variable

These should be specified by the programmer, and to this end, we can push and pop cleanup handlers (like `free`, `close`, `pthread_mutex_unlock`, etc.) into the cleanup stack as we want. To do this, we use:
```c
/**
 * push a routine into the stack
 * routine: pointer to handler
 * arg: arguments for the handler
 */
pthread_cleanup_push(
	(void) (*routine) (void *),
	void *arg
);

/**
 * execute: if 1, pop the handler AND execute it at the same time
 *          if 0, just pop it
 */
pthread_cleanup_pop(int execute);
```
###### Example:

Take a web crawler, we are gathering data and putting it in a list (possibly for hours), the code looks like this:
```c
list_item_t list_head;
void *gatherData(void *arg) {
	list_item_t *item;
	item = (list_item_t*) malloc(sizeof(list_item_t));
	getDataItem(&item->value);
	insert(item);
	printf("done\n");
	exit(0);
}
```
Now, imagine that we want to cancel this thread. Where can that happen? (i.e. what are valid cancelation points?)
- `malloc` is not a cancelation point
- `getDataItem` probably has numerous cancelation points in it
- `insert` is *not* a cancelation point (unless it's implemented really badly!)
- `printf` is basically just `write` to `stdout` and thus is a cancelation point

Therefore, it's quite likely that this code will cancel while executing `getDataItem`, but is that valid? Well, if you just exit in that function, you will never free what was allocated with `malloc`!
Thus, that is not a good cancelation point.

On the other hand, forcing it to happen in `printf` by disabling cancelation everywhere else is bad, since `getDataItem` may run for hours while it's result is not needed! **Cancelation should only be disabled for a very very short time.**

The solution as you might have guessed, is to do `pthread_cleanup_push(free, item)`.  Now, even if we do end up canceling in `getDataItem`, we'll have no problems; right??

Well, not really.
The problem happens when we cancel in `printf`. Note that other threads may not have any idea about the cancelation of this thread, they may only see that something is in the list and go on to process it. To process that list, they will also probably call `free` on the item in the list and thus call `free` twice on the same thing, which causes a memory corruption bug!

The thing that we should also do, is to pop the cleanup handler without executing it after we exit `getDataItem` and before we call `printf`. This way, we'll have the desired data item present when we cancel and other threads may attempt to process it and free it themselves with no issue.

>[!BUG] Common Bug
>The push and pop functions should never be called in separate functions, they should be called in the same function. If not, weird compile time error may happen.

###### Canceling In `pthread_cond_wait`

This is a weird case, if you are executing a guarded command, you have something like:
```c
pthread_mutex_lock(&m);
while (!guard) {
	pthread_cond_wait(&cv, &m);
}
// critical stuff ...
pthread_mutex_unlock(&m);
```
As we said, `pthread_cond_wait` is (and should be!) a cancelation point, but what will you do with the mutex?
You might say that "well, the mutex was locked! just push `pthread_mutex_unlock` into the cleanup stack and call it a day!".

Well no. The problem is that `pthread_cond_wait` NEEDS to unlock the mutex towards the end, before it goes to sleep, however, we have no idea if that sleep is actually the only point in this function where we can cancel! (at least, the implementation does not demand it!)
- If `pthread_cond_wait` is just at the beginning of it's execution, then yes, the mutex will be locked when we cancel and thus cleanup with `pthread_mutex_unlock` will work.
- If `pthread_cond_wait` tries to sleep and *then* cancels while calling `sleep`, then the mutex was already unlocked, and we end up calling `pthread_mutex_unlock` twice! 
  This can cause a severe memory corruption bug later!

It turns out, based on implementation, there really is no way to say whether or not after cancelation, the mutex is locked or not! Here, we are at the mercy of the library programmers, and this is one of the key things to consider when you use ANY threading library.

>[!IMPORTANT] POSIX Convention
>If cancelation happens during `pthread_cond_wait`, POSIX guarantees that the mutex will be **locked.**
>Thus, it is correct to push `phread_mutex_unlock` into the cleanup stack.

>[!NOTE] Cancelation In C++
> Up until recently, C++ flat out did not support thread cancelation! All C++ threads were required to self terminate.
> 
> Now you can do it, but there is one thing that you must consider, and that's cleaning up object destructors. 
> 
> By default, C++ compilers always generate code at the end of a function to automatically destroy local objects, but **don't necessarily do that for cancelation!** It depends purely on the compiler, you should read the man page and see for yourself. 
> Do not ignore this, since if you decide to do it yourself by pushing it into the cleanup stack, you may end up destroying an object twice, and you will either SEGFAULT or get a memory corruption bug.
