
**Paper:** [Achieving Microsecond-Scale Resource Fungibility with Logical Processes](https://www.usenix.org/system/files/nsdi23-ruan.pdf)

# Introduction

For a very long time, CPU was one of the most costly components in a computing system, but as of now, memory has quickly take over the cost of CPU, closely behind GPUs. 
The main reason that the DRAM demand went up was because of Big Data computations.
- In Big Data, you want to keep data in-memory as long as possible.
- The cost of the same unit of DRAM is not dropping significantly anymore.

Thus, the more the demand goes up, the more the *actual* cost of purchasing them goes up, and this is one of the main reasons that we still have 8-16 GB memory in virtually all smartphones and most consumer machines.

## Sources of Inefficiency in DCs

- Many DCs are heavily overprovisioned, even though they take advantage of statistical multiplexing.
- Resource requirements are multi-dimensional (i.e. a job needs a certain amount of memory, **and** CPU **and** GPU **and** etc.) and if any of them cannot be satisfied in an instance, then that instance cannot be used.

The second point is of particular interest to us, since it causes resources to become *stranded*, meaning that they remain unused, despite being there:

![[Pasted image 20241203052058.png|500]]

In the figure above, our instance requires a certain amount of CPU and memory to execute, but there is no machine that has both at the same time, despite the fact that the DC as a whole does, and so we end up being unable to submit this instance.

This means that a whole lot of resources are always stranded in the DC, being completely free, but unable to be used for different jobs.
Memory is the main bottleneck here, since we cannot make it as abundant as we wanted, so what we need to do, is to resolve this pressure of memory.

One of the main ways that we do that, is *disaggregation*.

## Hardware Resource Disaggregation

The idea is:
- Connect a bunch of machines, and share their resources in a pool (i.e. CPU, memory, etc.)
- Any time some machine needs some compute, we just pick resources from the pool

![[Pasted image 20241203052334.png|500]]

The price being paid here is:
- **Latency:** If you fetch an object from some remote memory, you incur a cost of waiting for network.
- **Non-Locality:** We no longer have clean control about where resources are placed.
- **Complexity:** You must keep track of where each object is and what it is doing.

This becomes prohibitive at scale (especially in terms of latency), so another simpler way to handle this is *migration*.

Migration means to transfer a live process (or VM) from some instance to another with higher resources. We saw how this works in CS656, but the main thing to consider here is that it is a costly process, and needs to take a long time if we want to make sure that we are not disrupting the service.

There are two ways to do this:
- **Coarse Grained:** Migrate entire processes and VMs between instances.
  Easy to do, but very costly in terms of performance and can still waste resource.
- **Fine Grained:** Migrate small pieces of each application to different instances.
  Very complex to implement, but much more resource efficient.

This bares some resemblance with how *microservices* work, where we break down a monolithic application into many small pieces and scale them independently as needed.

![[Pasted image 20241112124045.png|400]]

# Nu

We want to implement fine migration, but with lower complexity, and in order to do that, we use a simple abstraction on top of OS processes.
The idea is to break down processes into smaller *proclets* (which we will discuss how it is done), and migrate them quickly and easily.

>[!IMPORTANT] The general idea is:
>- Make process resource assignment really fine grained. The benefit would be that we can minimize or outright eliminate any stranded resource in the DC.
>- Make fine-grained process migration fast. The benefit would be that we no longer would have to overprovision so much for everything.

Before we talk about the *Proclets* in Nu, we should talk about *logical processes*.

## Logical Processes

![[Pasted image 20241112124616.png|500]]

A *Logical Process (LP)* is a process made up of smaller atomic units of memory and CPU (i.e. smaller heap and stack and threads), which the process uses to do function calls and computation.
A LP can span across multiple machines, since it has its own heap and thread.

Let's have a quick peek at Nu, which implements proclets:
```cpp
struct Accumulator { 
	Accumulator (int val) : val_(val) {}
	void Add(int n) {
		std :: scoped_lock l(mu_);
		val_ += n;
	}
	int Get () {
		std :: scoped_lock l(mu_);
		return val_;
	}
	std :: mutex mu_;
	int val_;
};

void mainfunc () {
	// Creates two proclets with root class Accumulator. 
	auto p1 = make_proclet<Accumulator>(10);
	auto p2 = make_proclet<Accumulator>(10);
	/**
	 * Invokes Get() on p1; prints 10. 
	 * This function reference to `Get` works since all instances have the
	 * same binary file loaded in memory.
	 */
	std :: cout << p1.Run (& Accumulator :: Get);
	// Invokes a closure on p1; prints 15. 
	std :: cout << p1.Run (
		+[](Accumulator &a) {
			a.Add (5);
			return a.Get ();
		}
	);
	// Invokes Get() asynchronously; prints 25. 
	auto f1 = p1.RunAsync (& Accumulator :: Get);
	auto f2 = p2.RunAsync (& Accumulator :: Get);
	std :: cout << f1.get () + f2.get ();
	// Adds p2’s value to p1 by invoking a closure on p1.
	p1.Run (
		+[]( Accumulator &a, proclet p) {
			auto v = p.Run (& Accumulator :: Get);
			a.Add(v);
		}, p2
	);
	// Arguments statically type checked; DOESN’T COMPILE!
	// p1.Run(&Accumulator::Add, 10, 20);
	// Proclets are freed when mainfunc() gets out of scope
}
```

There are limitations to this:
- All proclets communicate only using method invocations.
- No two proclets can share any memory with each other (this simplifies things)

Nu automatically converts method invocation on remote proclets to RPCs, but is smart enough to prevent that if it realizes that the proclets are co-located (i.e. on the same machine), and in that particular case, just calls the function directly.

![[Pasted image 20241203054724.png|600]]

Nu uses a centralized controller that keeps track of where each of the proclets live, and the runtime on each machine will cache result locations so that they won't have to bother the controller for each invocation.

For the case where a machine is becoming overloaded, each machine runtime would first report overload to the controller, and the controller would then tell it where to migrate to.
The key here is very similar to what happens in the Global Scheduler for Ray, we do not want the controller to be bothered with IO handling to migrate data, we only want to be busy monitoring things around it so that it make good decisions when it counts.

As such, the actual migration task is coordinated among the source and destination runtimes, not the controller itself.

![[Pasted image 20241112131141.png|500]]

From the point of view of virtual memory, Nu never touches the text and global read-only data, and that will look exactly the same in the virtual memory map across all instances running a process. Proclets however will have their own heap, stack and threads and have them distributed arbitrarily across machines.

## Network and I/O

LPs will handle IO between them through the runtime (which provides a wrapper around TCP connections maintained by Nu), but any other IO operation would have to be done "traditionally", and this might necessitate that some of the proclets would have to use completely local resources (like say a very large file), and for these scenarios, developers need to *pin* a process and its proclets to a particular machine to prevent Nu from migrating them around.

## Memory Management

Proclets are assigned heaps via a custom Slab Allocator that hijacks calls to `malloc/calloc` or other allocators, and makes sure that calls pointing to a particular heap get directed to the correct proclets.

Another thing to consider also is that Nu implements some simple amount of Fault Tolerance by using Primary-Backup replication and keep a backup of particular Proclets in certain places.
It is important to stress that this makes things hard, since we need to filter out duplicates (this is similar to VM-FT, where we had a backup of the VM that received the same inputs, but we just ignored the outputs). Nu uses a RIFL-like mechanism that does that and makes sure that calls are done exactly-once.
