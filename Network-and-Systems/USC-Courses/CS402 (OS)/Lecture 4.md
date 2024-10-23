## POSIX Threads (cont.)

Recall:
- Threads share the same code, data, heap and BSS spaces.
- Threads need separate stack spaces.
- The first stack frame of each thread, is the starting routine of the thread, with a single argument.

>[!BUG] Common Bug
>Be careful about passing arguments. You should allocate memory for the argument and THEN pass it to the thread if you want to use multiple threads.
>Inside a thread, you should also be careful about the lifetime of your variables. If you also manually allocate your threads, you should also care to free them.
>What this means, is that if you prefer to instead of:
>```c
>pthread_t thread;
>pthread_create(&thread, ...);
>``` 
>To write:
>```c
>pthread_t *threadPtr = (pthread_t*) malloc(sizeof(pthread_t));
>pthread_create(threadPtr, ...);
>```
>Then you must also do `free(threadPtr)` when you are done.
>This not only prevents memory leaks, it also prevents *memory corruption bugs*, which happen when a memory is reused unexpectedly (i.e. you rewrite to your own memory without knowing it).

### Keeping Child Threads Alive

Once a function returns, all it's local variables are considered garbage. Therefore, when the function creating the threads returns, all of it's threads may die with it as well (they may not, if for example, nothing in the stack space changes after the function returns, but it is still considered garbage memory).

One of the ways that you can handle this is to wait for each thread (or a specific thread) to die, similar to how we dealt with processes and their children. To do this, we can use a function that implements something similar to `wait`, but for threads. However, we'll deal with that later, for now we introduce `pthread_join`:
```c
int pthread_join(thread_t thread, (void **) ret_value);
```
Here, the second argument is the return value of the thread, and is supposed to be the address of a `void*` variable.

So now, you can do:
```c
typedef struct {
	int first, second
} two_int_t;

void *incoming(void *arg) {
	two_ints_t *p = (two_ints_t*) arg;
	// code ...
	return NULL;
}

void *outgoing(void *arg) {
	two_ints_t *p = (two_ints_t*) arg;
	// code ...
	return NULL;
}

rlogind(int r_in, int r_out, int l_in, int l_out) {
	pthread_t thread_in, thread_out;
	two_int_t in = {r_in, l_out}, out = {l_in, r_out};
	pthread_create(&thread_in, 0, incoming, &in);
	pthread_create(&thread_out, 0, outgoing, &out);
	/* This call is blocking.
	 * We don't care about the return values.
	 **/
	pthread_join(thread_in, 0);
	pthread_join(thread_out, 0);
}
```
### Thread Termination

To terminate a thread, there are two ways:
- Call `pthread_exit` in any function in the thread. Doing so like `pthread_exit(void *ret_value)` will kill the thread and set the return value into the thread's Thread Control Block for the parent thread to use.
- Return from the initial procedure. For example, a thread can return with `return ((void*) 2)`. To see how this works, remember that the thread isn't the only thing that is pushed into the stack, there is also a thread initializer that is called first. The initializer will receive this return value, and then call `pthread_exit` on it like above.

>[!FAQ]
>The Thread Control Block, while looks like it is a kernel data structure, may actually not be a kernel data structure in practice. This is the main difference between a user thread and kernel thread. As we said however, in this class, we consider the two indistinguishable, and we implement the control block as part of the kernel.

>[!IMPORTANT]
>If a thread in a process calls the `exit` Syscall, **the entire process and it's threads will terminate**.

In general, care should be taken about terminating a thread. Here is an example:
```c
int main(int argc, char *argv) {
	// Create 100 threads
	return 0;
} 
```
Here, `main` returns after requesting the thread creation, the entire process then exits and all threads will be destroyed. However, if you were to ask *how many of the 100 threads were run*, the answer would be **we don't know**, all of them might be finished by the time we return, or none of them. This code may exhibit unpredictable behavior.

On the other hand, if you were to do:
```c
int main(int argc, char *argv) {
	// Create 100 threads
	pthread_exit(0);
	return 0;
}
```
This will terminate the main thread, but never calls `exit`. This means that the created threads are left alone and the process will not return. This is bad, and thus, most implementations handle this case for you and assume you *forgot* to return correctly.

In particular, POSIX counts the number of threads in a process, and once main thread exits, it will check if the number of active threads is 0 or not. If it is not 0, the library immediately calls `exit` with some arbitrary return value and kills your process.

Therefore, unless you have good reason and understand what you do, **you should join your threads**, this at least makes sure that your code exhibits predictable behaviors.

>[!IMPORTANT] Some Clarifications
>We have been using the parent/child abstraction for threads like processes, however, this really isn't true. Threads, unlike processes, are *peers*, in fact, any thread may decide to join any other thread and wait for it to return.
>
>In light of this, the following bare mention:
>- Similar to processes, if a thread terminates, but was never joined with another thread, it will become a *zombie*. The library will then free up all data about this thread, except it's thread identifier, it's return value and it's *stack space*. Why the stack space? Well, we need to call `pthread_exit` at some point to free this thread, but you can't call a function in an address space, **if you don't have a stack to begin with!**, therefore, it is crucial that we keep the stack of the thread as well, so we can call `pthread_exit` in it.
>  There are threads that are *detached*, in fact, in POSIX, you can specify a thread as detached by using `pthread_detach(pthread_t)`. Doing so, prevents anyone from joining to it and it will have to be cleaned up later by the library itself.
>- Do not join two threads to the same thread. Doing so might cause thread IDs to get reused, leading to undefined behaviors.

>[!Note] Thread Attributes
>As we said, we don't need thread attributes in this class, however, there is one that might be useful to some people's codes, and that is the thread stack size.
>In all operating systems, threads are given a default value for their stack size (in most Linux distros, that is usually around 8 MB), but:
>- If you have many threads, you might wish to lower this number.
>- If you have threads that run long recursive calls, you might want to increase this.
>This can be done in POSIX like the following:
>```c
>pthread_t thread;
>pthread_attr_t attr; // Create attribute object
>pthread_attr_init(&attr); // Initialize attribute object
>pthread_attr_setstacksize(&attr, 1024 * 1024 * 20); // Allocate 20 MB stack space to thread
>pthread_create(&thread, &attr, startroutine, arg); // Create thread
>pthread_attr_destroy(&attr); // Free attribute object to prevent memory leak
>```

### Example: Matrix Multiplication

You remember matrix multiplication, right?
Given $A \in \mathbb{R}^{m\times n}$, $B \in \mathbb{R}^{n\times k}$, the matrix $C = A \times B$ is defined as $C \in \mathbb{R}^{m \times k}$ such that $c_{il} = \sum_{j=1}^{j=n} a_{ij}.b_{jl}$. Multiplication can be done exactly like this, but with threads, we can parallelize it. On a single CPU, this does not help much, but on multiple CPUs, it might help drastically. To do this, one can write:

```c
int A[M][N];
int B[N][K];
int C[M][K];

void *matmul(void *arg) {
	int row = (int) arg, col;
	int j, sum;
	for (col = 0; col < K; col++) {
		sum = 0;
		for (j = 0; j < N; j++) {
			sum += A[row][j] * B[j][col];
		}
		C[row][col] = sum;
	}
	return 0;
}
```
And then, do:
```c
#include <stdio.h>
#include <string.h>
#include <pthread.h>

main() {
	int i;
	pthread_t threads[M];
	int error;
	/*
	 * Initialize matrices, allocate memory
	 * and other stuff.
	**/
	for (i = 0; i < M; i++) {
		if (error = pthread_create(
			&threads[i], 0, matmul, (void*) i
		)) {
			fprintf(stderr, "pthread create: %s", strerror(error));
			exit(-1);
		}
	}
	// Wait for everything to finish
	for (i = 0; i < M; i++) {
		pthread_join(threads[i], 0);
	}
}
```
If you write this in `mat.c`, and want to output the executable `mat`, you should do:
```cmd
gcc -o mat mat.c -pthread
```
This compiles `mat.c` and then links it with the POSIX thread library so that the functions used above will be in the code section of the address space.

## Synchronization

Now, we come to the biggest challenge of multi-threaded programming. The goal is to prevent threads from having undesirable effects on each other. The term "synchronization" here does not mean to do things at the same time as the dictionary might suggest, it means the opposite in CS, to *prevent things from happening at the same time*.

This, ties synchronization in this context to the notion of **Mutual Exclusion**, which is the main way that we implement this concept. Checkout the article on [Therac-25](https://en.wikipedia.org/wiki/Therac-25) to see how not doing these things correctly can be disastrous.

### Example

To see how these errors may manifest, consider the two following threads:
```c
Thread1:                    Thread2:
x = x + 1;                  x = x + 1;
```
On paper, the order of execution between these threads, has no effect on the final value of x. If x started from 5, it will end up on 7, regardless of which thread executes first.
However, **this is not the case**.

To see why, please remember that you should consider the *low level language* during execution, not the high level language. The low level equivalent of the above would be:
```c
Thread1:                    Thread2:
load r1, x                  load r1, x
add r1, 1                   add r1, 1
write r1, x                 write r1, x
```
Assume that we run the first two instructions in `Thread1`, then we switch to `Thread2` after saving the context of `Thread1` (i.e. when we return to `Thread1`, we have the value of 6 in `r1` and we start from the 3rd line).

In `Thread2` we will execute all instructions, which leaves us with writing the value of **6** into the memory location of `x`. Now we return to `Thread1`, the context is restored and the register `r1` now contains the value 6, finishing the 3rd line puts the value **6** into the memory location `x`. 

The program now finishes with the value of `x` being 6!

You can verify that whether or not the two threads run on different CPUs, has no effects on the above execution.

The problem here is that high level code looks like it runs with a single instruction, i.e. it's *atomic*, there is no way that these instructions execute half-baked. This is clearly not true in a multi-threaded environment, unless **all interrupts are disabled**, which is not a good idea!

If you disable interrupts, a lot of bad things can happen, what if your thread runs into a problem that never allows it to terminate? Disabling interrupts is **NOT** the solution here.

>[!IMPORTANT] Atomicity In Operating Systems
>An atomic instruction means that it allows no changes to any variable involved in it's execution. 
>So, an atomic instruction *can still be interrupted by things that have nothing to do with it*. So, the CPU can stop executing atomic threads and go handle some random hardware interrupt, and then come back, and still keep the thread safe and atomic from a high level view.

### Locks

One abstraction that solves the problem of atomicity is to put all variables of a thread in some "safe deposit box", and lock it. What that means is that as long as a thread has this box opened, no other thread may attempt to open it, and will have to wait.

In order to keep things from sleeping for too long, we also apply the constrain that the thread currently using the box is not allowed to go to sleep. The operation of accessing the box itself needs to be atomic as well (there is chicken or egg problem here that we'll discuss later!).

In the CS jargon, we say that ***all variables in the box can be accessed atomically, with respect to the operation of the box***, this is just a fancy way of saying that as long as you use the operations of the box to touch these variables (read/write), all operations that you perform on these variables will remain atomic.

One of the most widely used implementations of this safety deposit box, is the **mutex** lock. The mutex lock in POSIX is a data structure of the type `pthread_mutex_t`. Once a lock `m` is initialized, it can be used like the following:
```c
int x; // Some variable that we want to keep safe

pthread_mutex_lock(&m);
// Do things with x
pthread_mutex_unlock(&m);
```
In the CS jargon, any code that uses these protected variables is called a **critical section**. To implement the mutex abstraction, **all critical sections need to begin with a lock operation and end with an unlock operation**. We will always implement threads this way, and therefore the code is critical *with respect to the mutex `m`*.

>[!IMPORTANT] Summary
>- Code that runs between the lock and unlock operations of a mutex, are called **critical sections with respect to that mutex**.
>- The system guarantees that all code that is a critical section with respect to the same lock, execute only one at a time. This is the implementation of mutual exclusion for high level languages.

### POSIX Mutex

POSIX Mutex uses 4 fundamental operations:
- An initializer `pthread_mutex_init(pthread_mutex_t*, pthread_mutexattr_t*)` to create a mutex structure **after dynamic** allocation.
- A destructor `pthread_mutex_destroy(pthread_mutex_t*)` to destroy the mutex structure **before** it is freed with `free()`.
- A **blocking** lock operation `pthread_mutex_lock(8pthread_mutex_t)`.
- An unlock operation `pthread_mutex_unlock(8pthread_mutex_t)`.

Initialization can also be done with the macro `PTHREAD_MUTEX_INITIALIZER`, so you can write:
```c
pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;
```
>[!FAQ] What Do We Initialize
>POSIX Mutex can block calling threads by putting them in a queue, if the mutex was already locked when the thread called `pthread_mutex_lock` on it. Doing this, will cause the thread to cease execution and go to sleep until it comes on to the head of the queue.
>
>This list can be seen as just our usual linked list, and thus the initialization operation will initialize this list. On the other hand, the destructor will clear this list and prevents threads from sleeping forever. This is why you should use the destructor **before** you use `free()` on it.

It is important that you know that while this solution guarantees safety, it does not guarantee *liveness* by itself. What this means, is that there are situations where all threads may get permanently blocked. 

One concrete example can be seen when using **2 locks**:
```c
void proc1() {                              void proc2() {
	pthread_mutex_lock(&m1);                    pthread_mutex_lock(&m2);
	// Do things with objects critcial          // Do things with objects critical
	// with respect to m1                       // with respect to m2
	pthread_mutex_lock(&m2);                    pthread_mutex_lock(&m1);
	// Do things with objects critical          // Do things with objects critical
	// with respect to m2 AND m1                // with respect to m1 AND m2
	...                                         ...
	// Release in reverse order                 // Release in reverse order
	pthread_mutex_unlock(&m2);                  pthread_mutex_unlock(&m1);
	pthread_mutex_unlock(&m1);                  pthread_mutex_unlock(&m2);
}                                           }
```
If the processes are given to separate threads, process 1 might lock mutex 1 and then try to lock on mutex 2, but get blocked. Later, process 2 might acquire the mutex lock 2, and proceed to lock on mutex 1, however, mutex 1 is in the hands of process 1, which is waiting for process 2 to release the lock! The two processes will get stuck in a **deadlock**.

At this point, the only solution is to kill the program.

Deadlocks can be visualized with a *wait-for* graph. The nodes of the graph are all the active mutex locks in the system, and mutex `m1` will have a directed edge to mutex `m2`, if and only if a thread attempts to lock mutex `m2` while having mutex `m1`.

If this graph contains a cycle, an execution *might* exist that causes the programs to go into a deadlock and never wake up. We say "might", because the graph usually does not consider how the programmer sequences the threads.

For the above program, the graph looks like the following:

![[Pasted image 20230206004547.png]]


