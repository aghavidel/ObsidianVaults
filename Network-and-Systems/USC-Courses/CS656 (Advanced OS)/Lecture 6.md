# Race Conditions

Until now, we blamed everything on the hardware. Most of the problems that we discussed were because of hardware errors, failures, etc.
Today, it's all about the developers! We are going to discuss race conditions.

## Intro.
Just the good old "two threads doing x = x+1 without locks" example. Without locks, you will get 2 different values depending on what the kernel scheduler does. 
The main reason of why this happens, is because CPU instruction sets only give you very low level atomic operations (i.e. storing and reading data to memory is atomic, not increment)

Usually, you will need to use Mutex and Condition Varuables to make sure that only a single thread uses shaed variables without getting stuck in a critical section.
Mostly, we also use the Monitor abstraction, where the programming alngauge provides a construct that has these condition variables encapsulated and will help the programmer avoid race conditions.

So why aren't we done?
Well, one big problem is that this clean way of protecting variables only works if the programmer already knows what variables need protecting, but:
- What if the variable is not defind during compile time?
- What if the variables are only created during runtime? (e.g. heap variables)

## How To Describe Multi-Threaded Programs
One way of formalizing multi-threaded executions is with Lamport's "Happens-Before" relationship. We say $a \rightarrow b$ if every event that preceeded $a$ is already done, and $b$ has not yet happened when $a$ is done.

This relation is transitive, and as such we can chain it. 
A race condition happens when there is no plausible "Happens Before" relationship between accesses to the same variable.

While we can use this to get the relation over a large program, the problem is that actually being able to see this race condition happen, depends on the thread scheduler. So how we can get rid of that?

## Eraser ...
We associate a Mutex lock to every shared state variable. Define $C(v)$ to tbe the set of all locks that protect the variable $v$. We call $C(v)$ the **LockSet** of the variable $v$.

The LockSet algorithm works by initializing $C(v)$ and then at each cod line, takes the intersection of $C(v)$ with the set of locks held during that point. If the set turns out to be empty, we scream at the programmer.

We want to execute this algorithm on an arbitrary binary file, and to do that, we keep track of every single memory access and mutex and then create the LockSets.
It is important to note that by definition, that is a huge aount of overhead, but luckipky, the LockSt tends to be pretty small, and instead of mapping memory locations to locksets we just keep the indices to the locks in the lockset. 
You can also use a Hash Table to speed up lookups.

So during runtime, when we access any memory location, we check whehter t not we have its lock, then we step the LockSet algorithm and we get a null set at any point, we scream.
This still has the problem of running very slowly though, so we can't ship programs with this!

## Insights

One of the main things is that the LockSet algorithm can produce a lot of false positives, since the set of variables isn't clear during runtime.
There is also a lot of nitty-gritties here, like with the current naive version of LockSet, if some one initializes a variable for te first time, since LockSet sees a meory access while the LockSet is empty, it is going to complain without reason!

So how do we fix this? **We only refine LockSet when a second thread wrtes to a variable**. This helps prevent False Alarms, but it also means that the algorithm depends on the sceduler for being able to detect races a lot more.

Even with this though, we can still have false alarms, since programmers try to minimze lock use to improve performance. See page 18 of the lecutre for example.
If we move teh whole IF block into the lock-unlock, eveyr thread that acceses the file pointer will have to get the lock, but if we mov the if conditions out of the locking, then only the first thread that sees the file pointer is null will have to execute te section.

With all of this though, Eraser will still complain! There are many valid cases where a programmer purposfully wants to avoid locks, which looks shifty, but is actually safe!

Also remeber, whenever we have multiple locks in a program, you MUST have a global ordering of locks to make sure you acquire them in a correct order. See the example on page 19 for example of deadlock!

