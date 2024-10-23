**Paper:** [Horcrux](https://www.usenix.org/system/files/osdi21-mardani.pdf)
# Automating Parallelism

Harking back to our discussion with Coz, we know that we almost always use threaded programs these days. But it turns out that not all code needs to be threaded!
- Sometimes, it is just not needed. Not having to go through the mental jugglery of threaded programs is nice
- If we need predictable and very safe programs, threads start to become less attractive

## Setting

One particular front of technology that are still doing things single threaded are JavaScript browser codes. These things do not usually take advantage of multiple CPU cores very well. This means that in their current form, throwing more cores at them just won't help.

You can see this here:

![[Pasted image 20240415001919.png|500]]

Recall that JavaScript is fundamentally single threaded, and developers have been coding that for decades now. So it is important to note that:
- JavaScript code is not *meant* to be multi-threaded, you can't force it to be multi-threaded
- Some JavaScript code on the browser does very little, and spawning a whole thread just for that is overkill 

So, we can't go for naïve solutions like executing all scripts in a separate thread. We need to know what scripts are completely disjoint from each other and can be safely run in parallel.

>[!EXAMPLE]
>There are many scripts that modify shared resources, and usually they are executed in a *pipeline*, so breaking it into several parts and doing it in parallel without synchronization will create a completely different pipeline.
>Thus, we should avoid this type of solution.

## Solution

The solution would be like this:
- Break down each script to a function chain. Call the entry point for the execution the **Root Function**.
- Analyze the chain, and see what sort of global variables it modifies or accesses. Call the set of all of these variables the **Dependency Set**.
- For each page, create the set of root functions and their dependency sets. Find a minimum partition of dependency sets that are disjoint, and then execute the root function of each set in a separate core.

There is a problem though, JavaScript is, well, *JavaScript*. It is not strongly typed, it can evaluate anything you pass it on a single line and can use a ridiculous number of APIs. Scanning JavaScript code from head to toe is a losing battle.

How about analyzing the page tables during execution?
- Some JS code is non-deterministic by nature or depends heavily on input
- Control flow on the client side might be different from the analysis, since again, JS code is very malleable 

## Concolic Execution

We won't dive into this thing in detail, but just know that a Concolic Execution of some program, creates a symbolic tree of it that:
- Reasons about every possible control flow
- Shows the "conservative" execution of the program

"MAYBE SEE DETAILS IN THE PAPER"

## Evaluation

The main metrics are:
- The page load time. That is the main problem that we want to solve.
- The speed index (SI), which means how fast does the majority of the *visible* parts of a page load (i.e. kind of how long it takes for the frontend to load).
- Total computation time (TCT)

![[Pasted image 20240304154351.png|500]]

Wi-Fi performs a bit better since in these tests, it was faster, so it saw a greater improvement.
Since LTE is slower, then it becomes the bottleneck and sees less improvement.

![[Pasted image 20240304154614.png|500]]

Two things to note:
- Total computation had bigger improvement. This is because page load most likely needs to do some IO and that part can't be optimized much.
- Overtime, speedup goes down. This populates the cache and as such the speedup benefits go down, even though total execution time is lower!
