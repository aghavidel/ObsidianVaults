**Paper:** [Coz](https://dl.acm.org/doi/pdf/10.1145/3205911)
# Profiling Multi-Threaded Programs

Multi-threaded code is now a staple of most networking/cloud application, yet any time we write a program, it is not immediately obvious where they should be used and how. Many generations of programmers have learned the hard way that these programs are very tricky to write.

## Main Idea

One tool that programmers utilize to see what is actually worth speeding up and how their program is actually behaving is *profilers*. These tell you during runtime:
- How many time a function was executed
- How much the total runtime took

This is valuable insight and helps programmers see how things are behaving, but the problem is that we still need to decide *what function to speed up?*.
This is a delicate problem, since:
- It is not obvious *how* a function can be sped up
- Speeding up a function can actually increase contention over resources and *slow down* overall execution
- Most importantly, it may not even work! Since the function, while heavy duty, may not even rely on the critical path

>[!EXAMPLE] 
>The authors give an interesting example. Assume you are downloading a file. Usually when the file is being downloaded, a small code is called that draws a little animation while all of this is happening.
>
>If you profile this code, you would see that both the download function and the animation function run for the same time. Which one do you optimize?
>If you ignore context, and make the mistake of optimizing the animation function, then congratulation! You have successfully wasted your time!

Here is another example:

>[!EXAMPLE]
>Take the following code:
>```c
>void a() {
>	for (volatile size_t x=0; x<2000000; x++) {}
>}
>void b() {
>	for (volatile size_t x=0; x<1900000; x++) {}
>}
>int main() {
>	thread_t a_thread(a), b_thread(b);
>	a_thread.join();
>	b_thread.join();
>}
>```
>
>Here:
>- Optimizing `b()` does nothing even though it is taking a lot of time.
>- Optimizing `a()`, even significantly by itself, provides a measly 5 percent speedup at most!
>  
> The trouble is that `a()` is the one that determines the program execution time, but optimizing it is only beneficial if `b()` gets the same treatment as well!

To make sense of this, we introduce the notion of program *Execution Flow* and *Critical Path*:

- Program execution flow is graph like below, where each branch signifies the startup of a thread, and a convergence signifies the thread finishing 
- Critical path is the particular branch of execution flow that determines the total execution time

![[Pasted image 20240228141942.png|500]]

Now, in this paper, the idea is very simple. First, we get the results of a normal profiler, and then, we reason about whether or not a function speedup helps by actually *slowing down other threads*:.

To make this a bit more formal, consider we have a function `foo` with execution time $T_{foo}$:
- **Execution 1 (Real Speedup):** Reduce runtime of `foo` to $T_{foo} - d$ but then **pause ALL threads** for  $d$.
- **Execution 2 (Virtual Speedup):** When `foo` runs, **pause ALL OTHER** threads for $d$.

These two programs will have the exact *same* runtime!

A bit more clearly:

![[Pasted image 20240304130611.png|500]]

Let us parse this figure:
- In (a), we have our baseline
- In (b), we speed up each execution of `f` by `d`. Since the critical path is still on $t_1$ during both execution, this yields a speed up of $n_f . d$ where $n_f$ is the number of `f` executions on the critical path (2 being here).
  Note that if we *keep* speeding up `f`, eventually `g` becomes the bottleneck and $t_2$ becomes the critical path!
- In (c), there is no speed up at all. What we do is that while `f` is starting to execute, we pause all *OTHER* threads for `d` seconds (this can include a thread executing `f` itself!). This gives an equivalent delay of the execution time by $n_f.d$.

This gives a way to actually check which functions are on the critical path so speeding them up will actually help!

## Evaluation

On the other hand, remember when we mentioned contention on Mutexes? It turns out that in this particular case, if you speed up some things, it can actually *Slow Down* the program! 
The authors observed this with SQLite:

![[Pasted image 20240304132753.png|500]]

From these graphs, we see that beyond a 25 percent speedup, the program execution speed actually *degrades*!
This strongly shows resource contention, and this true here, since these function calls are all tied to a Mutex lock.

>[!IMPORTANT]
>It is important to note that given an input AND an implementation, Coz only tells you what thing is worth improving, it won't tell you *how* to improve it!
>For example, the paper gives an example about a hash table, where Coz deduced that the link list traversal loop would benefit from speedup, but the first way that they did it did absolutely nothing!

>[!NOTE]
>A general limitation of this is that it really only works if the program is completely monolithic, and we are actually able to slow down the execution of all threads. This however may not be the case. For example, if a service is going to run on a device, you can't usually tell a device to slow down!
>This may also be the case for distributed systems, that may rely on remote micro-services for operation.

