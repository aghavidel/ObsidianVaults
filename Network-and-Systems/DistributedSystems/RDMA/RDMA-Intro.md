**Source:** [Web Article](https://elements.tv/blog/an-introduction-to-rdma-remote-direct-memory-access/)

Historically, I/O and the OS have been deeply intertwined. None have worked truly without the help of other, and all programmable IO usually relies on some kernel support to implement things like locking, congestion control and resource allocation.

The kernel however, is supported by the CPU, which is much less optimized for IO than it is optimized for number crunching.
For all computer engineers, the knowledge of how to best take advantage of interleaving IO while still doing computation is one of the main keys of achieving acceptable performance with distributed application. One most simple method is to use threads and asynchronous execution, where we spawn a concurrent stream to handle the IO and return to the computation at hand until that process is finished or we are forced to wait for its completion.

While this solution works well in achieving high utilization, it is still a fact that previous CPU cycles are going to be wasted on just *managing* all IO operations. Thus, a notion of *offloading* such operations directly to the IO appliance has always been attractive. If hardware offloading could be perfectly achieved, then the kernel can be left untouched for executing network operations and the CPU can keep on doing what it is actually good at.

We will focus on one of the more successful ways of achieving this goal, which is dubbed **Remote Direct Memory Access** or RDMA. 

## Background

Usually, whenever any IO operation needs to be interleaved with other processes, threads are the way to go. The programming of this can be arbitrarily tweaked (e.g. synchronous/asynchronous, other flavors, etc.). For heavy, long running tasks, multi-processing can also be used if the hardware for it exists.

Ignoring the at times, head-scratching complexity of some of these programs, at the end of the day, they are still relying on two things:
- Some form of kernel support (e.g. the scheduler for threads, either in user space or kernel space)
- Precious CPU cycles!

While this form of IO is most certainly very flexible and even convenient at times, it cannot be denied that one can be worried about how it performs under heavy load. The main reason that the use of threads is now widespread, is that they not only simplified concurrent programming, they also are really good at taking advantage of IO delays. Indeed, don't really speed up execution in a literal sense, they just allow the CPU to do actual work while waiting for IO operations to complete.

All of this however, relies on the *assumption* that IO takes reasonably long time compared to CPU processing time. This assumption is still quite true, and can always be true if the system is large enough and distributed, but the gap has been rapidly closing with the development of very fast storage devices, and most important of all, very fast and dedicated networking appliances.

Indeed, there have been systems that have tried to go as far as trading of storage space with networking infrastructure. The speed-up has been so dramatic that executing local operations at times can take longer than executing them remotely on beefier machines, since the network latency is shrinking very fast.

There is also the concern of how *predictable* these systems are, especially in terms of latency. Indeed, it has been a notoriously bad practice to do number-crunching and IO on the same device, since one of them is going to put strain on the other one no matter what. 
As such, there have been many small and big ideas that try to somewhat alleviate the CPU from this burden, and one of the bigger ones have been *Kernel Bypassing Mechanisms*. 

The name is quite literal here, it is essentially an attempt to implement IO operations with little to no support from the kernel at all. Hardware Offloading for example is one of the oldest ways of doing this, where dedicated hardware is used to do IO operations and optimized device drivers are used instead to support the kernel interface.

RDMA schemes are also quite old, dating back to the early 90s, with commercial deployment in the late 2000s and widespread use in DCs currently. 

## Some More Details

This section is taken from [lecture notes by Radhika Mittal over at UIUC](https://courses.engr.illinois.edu/ece598hpn/fa2020/slides/lect18-RDMA.pdf).

We first discuss the oldest (and the simplest) implementation of RDMA, [[InfiniBand]] (IB). 
This most likely needs its own notes, but in brief:
- InfiniBand is wholly different from Ethernet, which means that it implements an entire communication *fabric*, not just a protocol.
- For the same reason, InfiniBand can have bad interactions with IP networks.
- It provided the first programming interface for actually implementing RDMA, the **Verb API**
- It is implemented in the NICs, not the kernel stack and does not require any host OS support

![[Pasted image 20240225190047.png|350]]


RDMA uses a completely different connection abstraction, very different from the channel based TCP/IP model with its `send/rcv` API.
From a very high level, two endpoints communicating across an IB fabric can be abstracted like the following:

![[Pasted image 20240225190249.png|500]]

Let's explain this picture a bit:
- The way that RDMA manipulates data is by directly referencing the virtual addresses of the desired data, perhaps even keeping track of a whole VMA the same way that the VFS in the kernel does. This although has its challenges which we shall see soon.
- Naturally, to avoid having to go through the VFS, the NIC itself needs to keep track of the mapping between virtual and physical addresses. The entity that keeps track of this mapping is the ***Memory Translation and Protection*** (MTP) block. It will also protect this regions by making sure access is not violated with the semantics of a VFS.
- Each endpoint implements a ***Queue Pair*** (QP), two buffers that are used for sending/receiving packets from/to that endpoint (so there are two QPs for each point-to-point connection).
- A QP provides connection-oriented/connection-less semantics, as well options for reliable/unreliable transfer. A reliable, connection-oriented QP is essentially a very low level TCP implementation.
- Connection establishment requires out-of-band transactions and exchanges. Remote/local node IDs need be set up, as well as remote keys and QP IDs. They are *NOT* really part of the IB protocol, but are discussed in the specification.

Lets drop a few more names:

![[Pasted image 20240225191632.png|350]]

- **Work Request (WR):** A struct that contains the metadata required for an operation (these include QP keys, the base address and the length of the remote data, etc.). 
- **Work Queue Element (WQE):** These encapsulate WR structs as individual data packets, with some extra information that we don't discuss for now. It is these units of data that are pushed into the QP buffers.
- **Completion Queue (CQ):** An individual CQ is associated with each QP on each endpoint. It will contain a result element, a **Completion Queue Element (CQE)** when a remote operation finishes. An application can synchronize on these events by waiting on this queue. The IB API guarantees that the generation of each CQE would cause the associated WQE to expire.

RDMA semantics provide 4 operations:
- RDMA Read: Fetch a remote data instance 
	- **Operation Metadata:** Remote virtual address and key + Remote data length
- RDMA Write: Modify a remote data instance
	- **Operation Metadata:** Local virtual address and key + Remote virtual address and key + Length
- RDMA Atomic: Atomically fetch-add a remote data instance, or perform a Compare-and-Swap with a local instance.
	- **Operation Metadata:** Local virtual address and key + Remote virtual address and key + Operation type
- RDMA Send/Receive: Just normal data send/receive

It is this metadata that is put into a WR object, and is sent over the fabric during the execution of the operation.

Today, the more widespread implementations of RDMA are the following:
- IB, a low level and very fast implementation that requires dedicated hardware and can usually only be used with purpose-built clusters.
- [[RDMA over Converged Ethernet (RoCE)]]: An implementation of RDMA over plain Ethernet. It and its second iteration, RoCEv2 which implements this on top of an IP stack, are the most widespread implementation in DCs and clusters.

We now discuss 3 papers that make use of RDMA or discuss it. We think they are key papers in understanding the landscape. They would require their own notes of course, so we keep things simple here.

## FaRM: Fast Remote Memory

FaRM is a distributed KV store and graph representation database. Its main, and only, communication primitive is RDMA (RoCE in particular):
- RDMA Reads to access remote data instance directly
- RDMA Write for fast message passing

To get a sense of how dramatic the performance improvement can be, take a look at this graph:

![[Pasted image 20240225202843.png|500]]

Let's ignore the difference between RDMA and RDMA msg for now, but you can clearly see that TCP is getting destroyed to say the least.
More alarmingly, the latency is also displaying an even worse trend:

![[Pasted image 20240225203129.png|500]]

This is the true benefit of decoupling IO and number crunching, the CPU is just overwhelmed under heavy load, which is to be expected, but it is *NOT* expected that the utilization drops by 2 orders of magnitude!

It is important to note that since RDMA is not general purpose, there is a LOT of tuning that needs to be done in order to get acceptable performance. 


