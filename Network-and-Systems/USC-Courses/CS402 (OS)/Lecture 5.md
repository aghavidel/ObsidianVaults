## Deadlocks

Last lecture, we introduce the concept of deadlocks with mutex implementations. 

Deadlocks are:
- Very hard to detect:
	- Deciding whether or not an operation may lead to a deadlock can be NP hard depending on a system.
	- Deciding whether or not a deadlock has happened may not even be possible in some systems.
- Very **easy** to prevent with restrictions:
	- For a deadlock to possibly happen, all the following conditions need to be true (i.e. these are necessary):
		- **Bounded Resources:** Only a finite number of threads may access a resource (always true with a mutex)
		- **Wait for Resources:** Threads wait for resources to be freed, **without** releasing resources that they hold.
		- **No Preemption:** Resources may not be forcefully revoked from a thread.
		- **Circular Wait:** A cycle exists in the wait-for graph.
	 By preventing one of these conditions, we prevent deadlocks entirely.

### Lock Hierarchy

One way to prevent deadlocks is to prevent the **Circular Wait** condition from ever happening. Doing this would require eliminating all cycles in the wait-for graph.

There are multiple ways to do this, but one of them is to assign a number to each mutex called a **level**, and asserting that if a thread holds a set of mutexes, it is not allowed to wait for any mutex that has a lower number compared to it's current locks.

To see this, imagine with have a 4 level hierarchy like below:

![[Pasted image 20230206011142.png]]

The question of how we assign these levels to mutexes or which mutex we end up giving if there are multiple choices is a welcome one, but let's put that aside now. 
Assume we have a thread that currently owns locks with level 2 and 3. This thread can **only** wait on locks with levels 4 or higher.

![[Pasted image 20230206011306.png]]

This solution works beautifully, but assigning the levels as well as the very restrictive nature of the hierarchy can easily become a problem.

### Conditional Locking

Looking back at the necessary conditions, condition 4 we just discussed. Condition 1 is unavoidable, and condition 3 is OS dependent. All we have is condition 2, we need to allow some threads to release their resources (i.e. release their locks) if they *suspect* we might be heading towards a deadlock.

To do this, we propose the following; We implement a routine into the threading library that let's us **check** if we can immediately acquire a lock. If this does not happen, we release **ALL** of our previous locks, and try again from scratch.

In POSIX library, this is done with `pthread_mutex_trylock(pthread_mutex_t*)`. This routine will return 0 if the calling thread can lock the mutex and will atomically lock it immediately. If not, it will return something else, and this means that if this thread attempts to lock this mutex, it will go to sleep.

Therefore, we can modify one of our processes in the previous example (say, `proc2`) to do the following:
```c
void proc2() {
	while(1) {
		pthread_mutex_lock(&m2);
		if (!pthread_mutex_trylock(&m1)) {
			break;
		}
		pthread_mutex_unlock(&m2);
	}
	// Do things with critical objects of
	// locks m1 AND m2
	pthread_mutex_unlock(&m1);
	pthread_mutex_unlock(&m2);
}
```
This never deadlocks.

The problems are:
- The codes are no longer symmetrical, indicating that some threads have subtly higher priorities (above for example, process 1 will always win over process 2 when attempting to lock both locks).
- There is a lot of busy-waiting with that `while` loop!

>[!FAQ] Sharing Data Among Threads
>Threads cannot speak to each other with calls, as they cannot share stacks. They can only share variables in the data or heap segment.
>
>Read only variables do not need a lock, but read/write variables require a mutex shared among all the threads that attempt to use these variables at any time. In general, try to limit the number of mutex locks you use, as nested mutex locks are not only prone to bugs and deadlocks, they tend to exhibit poor performance, unless some advanced POSIX techniques are used.

>[!Note] Beyond Mutual Exclusion
>Mutex is generally an overkill for most problems. Mutex completely eliminates parallelism in name of safety, but, there are cases where we can get away with less restrictions, namely:
>- Threads share variables occasionally (e.g. the **producer-consumer** problem, or many problems involving matrix multiplications).
>- Threads might just want to **look** at some variables (e.g. the **reader-writer** problem).
>We'll discuss these later.

## Producer-Consumer Problem

![[Pasted image 20230206013902.png]]

Given a limited buffer, two threads (i.e. the producer and the consumer) will share this as a data structure. The producer puts things in this buffer, and the consumer will consume them later. In the real world, these threads are physically separate and thus exhibit no problems in terms of concurrency, but in software, the rate at which the producer and the consumer process things, is unknown. 

This means that:
- The producer might create so many items, that the buffer might overflow.
- The consumer might consume so many items, the the buffer might become empty.

Both of these cases, necessitate that we block the corresponding threads and let the other thread catch up (i.e. in the first case, let the consumer empty the buffer a bit, in the second case, let the producer add something to the buffer).

However, **most implementations tend to be fair**, and this means that it is not likely that either of the two conditions above happen with high probability, since the CPU tends to prioritize the threads equally, unless the user specifically said otherwise.

Thus, the interference between these two threads is minimal, and using a mutex is just too inefficient in this case. Of course, use of mutex produces a *correct* solution, but we can do better. Note that we only need synchronization when the consumer has emptied the buffer, or the producer has completely filled the buffer, therefore the majority of the time, no synchronization is necessary.

### Necessary Tangent: Guard Statements

To talk about threaded programming, one needs to be very precise about critical sections, and one of the ways that you can do this is with *Guard Statements*, courtesy of Edgar Dijkstra. 

Guard statements have the following syntax (note the **square bracket instead of the usual curly bracket**):
```java
when (guard) [
	/*
	 * Once 'guard' is true, execute all of
	 * thie code atomically
	**/
]
```
So, what this means?

> When 'guard' is evaluated to be true by a thread, it WILL execute the body of `when` (also known as the *command sequence*) **atomically**.

So in some sense, the evaluation of `guard` and the execution of `when` together is the critical section. Therefore, **the command sequence by itself is NOT the critical section**.

![[Pasted image 20230206184355.png]]

In the above abstract figure, atomic means that the evaluation and execution of command sequence is done in basically zero seconds (again, note that this is the abstraction, of course the implementation won't execute in 0 seconds!).

### Semaphores

A semaphore is a non-negative integer `S`, defined with two guarded operations `V` and `P`:
```java
P(S):
when (S > 0) [
	S = S - 1
]

V(S):
[ S = S + 1 ]
```
The `V` operation is always enabled and atomic.

#### Example: Binary Semaphores

Initialize the semaphore `S` to start from 1. Then:

```C
void oneAtATime() {
	P(S);
	// Atomic operation with respect to the semaphore S
	V(S);
}
```
Each time `P(S)` is called when `S = 0`, nothing happens, thus, the calling threads need to keep calling this function to manage to execute the operation inside the function. The key here is that **only one thread may execute the section inside the function at any time**.

So in this context, semaphores are basically just a mutex.

#### Example: Counting Semaphore

Initialize the semaphore `S` to start from `n`, where `n` is some positive integer.

```C
void nAtATime() {
	P(S);
	// Only N threads may be here at any time
	V(S);
}
```
This shows the main difference between semaphores and mutexes. While with a mutex, only the thread that locks the mutex may unlock it, but with a semaphore, a thread may 'lock' a semaphore, but another thread may 'unlock' it, letting another thread in. 

### Solving The Producer-Consumer Problem

We use semaphores instead of a mutex:
```C
Semaphore bufferIsEmpty = BUFFER_LEN;  // P on this blocks if buffer is empty
Semaphore bufferIsNotEmpty = 0;        // P on this blocks if buffer is not empty
int nextIn = 0;
int nextOut = 0;
```
Now let:
```C
void producer(char item) {                   void consumer() {
	P(bufferIsEmpty);                            char item;
	buf[nextIn] = item;                          P(bufferIsNotEmpty);
	nextIn = nextIn + 1;                         item = buf[nextOut];
	if (nextIn == BUFFER_LEN) {                  nextOut = nextOut + 1;
		nextIn = 0;                              if (nextOut == BUFFER_LEN) {
	}                                                nextOut = 0;
	V(bufferIsNotEmpty);                             }
}                                                V(bufferIsEmpty);
                                                 return item;
											}
```

This code guarantees correctness, and synchronizes the producer and the consumer when we reach to fringe cases (empty/full buffer), the semaphores will put the faster thread to sleep, and only wake them up when the buffer isn't empty/full.

>[!TLDR] Summary
>We say that a mutex provides *fine grain parallelism*, while semaphores provide *coarse grained parallelism*. Semaphores however are much less general purposed, and for the most part, are only useful for Producer-Consumer problems.

#### POSIX Semaphores

In POSIX, semaphores provide the following API:
```C
#include <semaphore.h>
sem_t semaphore;
int err;
int pshared = 0;              // Not shared among processes
int init_value = INIT_VALUE;  // Inital value of the semaphore

err = sem_init(&semaphore, pshared, init_value);  // Initializer
err = sem_destroy(&semaphore);                    // Destructor
err = sem_wait(&semaphore);                       // P operation
err = sem_trywait(&semaphore);                    // Conditional P operation
err = sem_post(&semaphore);                       // V operation
```
As for their implementation:

<table>
<tr>
<th>Guard Syntax</th>
<th>POSIX Implementation</th>
</tr>
<tr>
<td>
	<pre><code>
	when (S > 0) [
		S = S - 1;
	]
	</code></pre>
</td>
<td>
		<pre><code>
		while(1) {
			pthread_mutex_lock(&m);
			if (S > 0) {
				S = S - 1;
				pthread_mutex_unlock(&m);
				break;
			}
			pthread_mutex_unlock(&m);
		}
		</code></pre>
</td>
</tr>
<tr>
<td>
	<pre><code>
	[ S = S + 1; ]
	</code></pre>
</td>
<td>
		<pre><code>
		pthread_mutex_lock(&m);
		S = S + 1;
		pthread_mutex_unlock(&m);
		</code></pre>
</td>
</tr>
</table>

As you cam see in the implementation of the `P` operation, there is two critical sections that are intertwined with each other. This is why the guard statement defines critical section like the way it did.

>[!FAILURE] Problem With Above Code
>The code above uses *busy-waiting* by using the infinite while loop. This means that the thread never gives up the CPU while it waits for the guard value to become true. 
>
>The right way to wait for some value to change, is to:
>- Go to sleep
>- When someone changes the code, make sure they wake you up!
>
>This is called *condition signaling*, and it's what we'll do later!

## General Multi-Threaded Problems

As we discussed above, guard statements are the easiest way to specify the high level behavior of a multi-threaded code, but how would we implement them in general?

Basically, how would you implement the following:
```c
when (guard(x1, x2, ..., xn)) [
	statement(y1, y2, ..., ym);
]
```
Where `guard` and `statement` are functions of multiple variables.

One way is to define a mutex for the set of variables `x1, ..., xn` and `y1, ..., ym`, identify the critical sections of the functions `guard` and `statement`, and execute critical section codes sequentially.

This however, similar to what we did with semaphores, ends up with a lot of busy waiting!

### Condition Variables (CVs)

![[Pasted image 20230206212708.png]]

The above shows an implementation of a guard statement with a mutex `m` and a condition variable `CV`. Now, the CV is just a queue of threads (remember that a mutex was a queue of threads and a boolean variable indicating if it's locked or not).

The difference is that for a CV, the queue is specifically for sleeping threads waiting to evaluate the guard. The "condition" in the name of *condition variable* refers to the guard statement, which the threads wait on to evaluate and execute the critical sections.

Once a thread changes a value in the CV, the CV will wake up one of the sleeping threads and ask them to reevaluate the CV, and if now it's true, we proceed to do the operation, atomically. A bunch of rules are added to make sure that this wake up call is not lost on the threads.

The basic operations of a CV are:
- **Signal:** Wake up one sleeping thread, and put it on the mutex queue to reevaluate the guard statement.
- **Broadcast:** Wake up *ALL* sleeping threads, and put all of them on the mutex queue.
So "wake up" here is more precisely "ready to wake up with a mutex unlock".

**No guarantee** is given that the guard statement will be true when a thread does evaluate it after a wake up call. If you do evaluate the CV to false, you go to sleep again. If not, execute your code and **leave the CV queue**.

In POSIX, this is done like the following:
```c
pthread_cond_wait(
	pthread_cond_t *cv,
	pthread_mutex_t *mutex
)
```
Here are the rules:
- The mutex `mutex` **MUST** be locked when you call this command.
- The code will atomically:
	- Unlock `mutex`
	- Go to sleep in the CV queue
- When the event is signaled/broadcasted, the code will return with `mutex` ***locked***.

The signal/broadcast operations may only be done with threads that actually have the mutex:
```c
pthread_cond_broadcast(pthread_cond_t *cv);
pthread_cond_signal(pthread_cond_t *cv);
```
These two functions are very easy to implement with a linked list. Imagine that the mutex queue and the CV queue are both linked lists, then:
- To implement signal, unlink a thread from CV and append them to the mutex queue.
- To implement broadcast, loop while the CV is not empty, and unlink and then append a thread to  mutex queue.

The tricky part is the wait operation alone, since if the wait is not done atomically, it's possible that a thread will miss the event of the CV, and go to sleep forever.

This is not a deadlock, since the resource can be revoked, but it's called **race condition**, which is just a fancy way of saying it's a **bad, timing-dependent behavior**.

In POSIX, CVs can be initialized similar to mutexes:
```c
// Statically
pthread_cond_t cv = PTHREAD_COND_INITIALIZER;

// Dynamically
int pthread_cond_init (
	pthread_cond_t *cvp,
	pthread_condattr_t *attrp
)

int pthread_cond_destroy(
	pthread_cond_t *cvp
)
```
Therefore, to implement any guard statement, you may follow this template:

<table>
<tr>
<th>Guard Syntax</th>
<th>POSIX Implementation</th>
</tr>
<tr>
<td>
	<pre><code>
	when (guard) [
		statement 1;
		...
		statement n;
	]
	</code></pre>
</td>
<td>
		<pre><code>
		pthread_mutex_lock(&mutex);
		while(!guard) {
			pthread_cond_wait(
				&cv, &mutex
			);
		}
		statement 1;
		...
		statement n;
		pthread_mutex_unlock(&mutex);
		</code></pre>
</td>
</tr>
<tr>
<td>
	<pre><code>
	[ 
		/*
		 * Code modifying the guard
		 **/
	]
	</code></pre>
</td>
<td>
		<pre><code>
		pthread_mutex_lock(&mutex);
		/*
		 * Code modifying the guard
		 **/
		pthread_cond_broadcast(&cv);
		pthread_mutex_unlock(&mutex);
		</code></pre>
</td>
</tr>
</table>

