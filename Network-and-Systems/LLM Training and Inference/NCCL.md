**Paper:** [Demystifying NCCL: An In-depth Analysis of GPU Communication Protocols and Algorithms](https://arxiv.org/pdf/2507.04786v1)
Some extra things have been added by me from [NCCL Docs](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/).

This paper is a technical survey about how Nvidia Collective Communication Library (NCCL, pronounced as `nickel`) works. Unlike general purpose collective communication libraries like MPI, NCCL specifically targets GPU-to-GPU operation, and makes efficient use of the inter-connect fabric (things like [[InfiniBand]], NVLink, etc.).

To get a good sense of how NCCL works and its high level API, this paper condenses the NCCL documentation (plus some extra details when needed). It is *NOT* a coding tutorial in NCCL, rather it is there to give a good and accurate high-level view of how NCCL components interact and what that would mean for people building systems with them.

The paper is structured as follows:
1. **General Overview:** API structure, and communication channel management.
2. **Protocols:** Details of the communication protocols used by NCCL and transfer models.
3. **Collective Algorithms:** Detail of the algorithms that NCCL uses to schedule collective communication calls.

# NCCL Overview

We first begin by describe the API that NCCL provides. In this context, all API operations are to be performed through a *Communicator* object that NCCL owns. These objects, abstract away all the low-level details and leave the user with simple API calls.
For our intents and purposes, this API consists of the following:

- **Communicator Management:** Set of APIs that are dedicated with initializing, maintaining and clean-up of communicator objects.
	- `ncclCommInitAll`: Single process/Single-threaded initializer that allocates using all GPU nodes available through a single thread of execution.
	- `ncclCommInitRank`: For multi-threaded or multi-process environments. Each process must call this separately using a shared ID to allow for CC.
	- `ncclCommDestroy` and `ncclCommAbort` are cleanup operations. The `destory` call will free the communicator while allowing for pending operations to finish, while `abort` kills the communicator no questions asked.
- **Collective Communication:** Including `ncclAllGather`, `ncclAllReduce`, `ncclReduceScatter`, `ncclReduce`, etc.
- **Point-to-Point Communication:** Including simple direct communication between ranks. For example `ncclSend` and `ncclRecv`.
- **Group Calls:** These are functions that bracket multiple p2p or collective operations to be executed together. Group calls must exist between `ncclGroupStart` and `ncclGroupEnd` calls. Tasks within groups can be aggregated such that they can be called in a single launch operation, and also allow many other things.

## Collective Operations

The NCCL collective operations are very close to how they are defined in MPI. In particular, collective operations are done over devices with different *ranks*, but unlike MPI, device to rank mapping need not be one to one.

With that caveat aside, the main NCCL operations match the ones in MPI. In particular:

- **Scatter/Gather**, where chunks are scattered/gathered to/from other ranks from/to a *root* rank.
  
![[Pasted image 20260319232254.png|500]]
![[Pasted image 20260319232306.png|500]]

- **Reduce/Broadcast/AllReduce**, where chunks in different ranks are reduced into one device, or optionally broadcasted by copying the result in the root to other ranks. Semantically, `Reduce` followed by `Broadcast` is equivalent to `AllReduce`, however that would be extremely inefficient and prone to failure, as the root tank becomes a choke point. In practice, NCCL assumes a few things about how GPUs are connected (in particular, either a ring, tree or double tree topology) and implements `AllReduce` using the next two collective operations.
  
![[Pasted image 20260319233035.png|500]]

- **ReduceScatter**, which essentially does `Reduce`, but chunks of the data will end up in different ranks as appropriate. In particular, with $N$ ranks, and ranks/chunks indexed from $0$ to $N-1$ , the $i$-th chunk of the reduction will be located in rank $i$.

![[Pasted image 20260319233706.png|500]]

- **AllGather** will gather properly aligned chunks of data from all ranks, into all ranks.

![[Pasted image 20260319233755.png|500]]

>[!FAQ]- How `AllReduce` is actually implemented.
>`AllReduce` is implemented by calling `ReduceScatter` first to create associated chunks of the reduction, and then calling `AllGather` to make sure all ranks end up with the same view of the result.
>
>For example, on a GPU ring with 4 nodes:
>- **GPU 0 holds:** `[A0, B0, C0, D0]`
>- **GPU 1 holds:** `[A1, B1, C1, D1]`
>- **GPU 2 holds:** `[A2, B2, C2, D2]`
>- **GPU 3 holds:** `[A3, B3, C3, D3]`
>  
>  It will proceed first with a `ReduceScatter`:
>  1. Each GPU sends its current chunk to its successor while receiving one chunk from its predecessor.
>  2. Each GPU reduces the received chunk into its current chunk, updating it in-place.
>  3. This is repeated 3 times, until each GPU ends up with a chunk of reduced data. In the case above, we end up with `ΣD, ΣA, ΣB, ΣC` respectively for each GPU.
> 
> Now we end with `AllGather`, each GPU sends its latest chunk to its successor and receives one from its predecessor for 3 times.
> 
> Computation aside, this takes 6 rounds of parallel communication (in general, a ring of $N$ nodes needs $\mathcal{O}(N)$ operations, while trees can make do with $\mathcal{O}(\log N)$), similar to `Reduce` plus `Broadcast`, but the payload size at each step is $1/N$-th of that case!

- **All-to-All**, semantically, a matrix transpose across all ranks. By far the most difficult operation, as there are no reductions to help preserve bandwidth.

![[Pasted image 20260322110959.png|500]]

>[!NOTE] The headache of `All-to-All`
>`All-to-All` is the only collective that allows for global redistribution of data. Each GPU is able to share chunks to each other GPU simultaneously.
>This operation is actually really important for Mixture of Experts ([[MoE]]). As you might guess though, this operation would be a nightmare to do on rings or trees:
>
>- On a ring of $N$ nodes, the operation takes $N-1$ steps with full bandwidth and increasing latency (the first step shuffles data one hop away, the second step for two hops, and so on ...). Much bandwidth is spent just using GPU nodes as transit, as opposed to allow for compute.
>- On a tree, we still need $N-1$ steps, but the tail latency scales $\mathcal{O}(\log N)$ instead of $\mathcal{O}(N)$, BUT this only really works if we have full bisection bandwidth (so a Fat-Tree/Clos topology), otherwise it will fail miserably due to congestion on the spine.
>  
>  The high latency of `All-to-All` is one of the main motivations for using [[NVLink]], which basically means that the effective topology is fully-connected (everyone is just one hop away all the time).

## Communication Details

### Kernel Launches and Channels

NCCL launches kernels from the OS. A CPU thread thus needs to get involved. NCCL supports 3 strategies for binding CPUs to kernel launches. These are:
- *One CPU core per GPU:* Fine grain control. If individual cores have hardware affinities (like say, a NUMA assignment), this can come in handy by binding GPUs to specific CPUs such that these affinities are honored.
- *One CPU thread per GPU:* If the scale allows, this might be ideal as it reduces the overhead of memory operations between separate ranks.
- *One CPU thread for all GPUs:* Easy and deterministic. Useful for cases where kernel launches cannot become a bottleneck (either because they are going to take a long time, or the scale of the project is very small).

>[!REMINDER]- CUDA Architecture
>A GPU is made up of many Streaming Multiprocessor (SM) units. These are cores with many execution threads with a shared L1 cache and register file. SMs crunch though data in parallel and execute kernel code.
>Optionally, SMs can be grouped into Graphic Processing Clusters (GPCs). These GPCs are able to coordinate among each other to some degree (by default, only threads within the same SM can synchronize with each other).
>
>![[Pasted image 20260328183916.png|500]]

Passing data form CPU to GPU requires copying of a buffer. In lieu of direct memory access technologies like `GPUDirect`, the SMs need to execute `memcpy` calls to copy such buffers into their cache. If the buffers are very large, SMs can be overwhelmed and fail to execute kernel code in time.

For this reason, NCCL assigns separate communication channels to each SM. The number of channels (and SMs) is decided heuristically based on buffer size. Note that we cannot be too extreme here:
- Too few channels will keep SMs busy doing IO and lowers throughput.
- Too many channels will cause NCCL to schedule partially filled buffers which means that the cost of IO will dominate the cost of computation.

As an example, when NICs get involved, buffer sizes are usually capped to at least 512 KiB. If NCCL were to assign so many channels that the amount of data handed over to each buffer per channel is lower than this, then we are losing IO efficiency. 
When a communicator object is crated and GPUs have been assigned ranks, NCCL decides on an initial number of channels to pre-allocate. Then, for each collective operation, the NCCL runtime decides on what communication algorithm to use based on the available bandwidth, message size and topology, and then heuristically chooses the number of channels to use.

A logical topology is assigned to the ranks that determines how channels actually communicate. This is where ring and tree structures come in for GPU communication (and even double trees, if more bandwidth is needed).

### Communication Protocols

NCCL employs multiple comm. algorithms. Three of the well known ones are **Simple**, **Low Latency (LL)** and **LL128**. These protocols employ different methods to trade-off between latency and bandwidth.

![[Pasted image 20260329141136.png|500]]

The figure above shows the end-goal of each protocol in terms of latency and bandwidth efficiency.

- **Simple**: Send data in large chunks. Designed to utilize bandwidth using large messages. Synchronization is done using memory fences, which induces high overhead. The overhead will be amortized for larger messages but will be prohibitive for smaller ones and the latency will remain noticeable regardless of message size.

>[!FAQ] About Memory Fences in CUDA
>Memory fences are standard and widely used in CPU-land, but they are a nightmare in GPU related applications and CUDA when it comes to multiple GPUs.
>A fence operating over multiple GPUs would have to specifically wait for _all_ writes to finish over the fabric that connects each hop. Unlike in CPU land where that fabric is the L1/L2 cache, in GPU land, this fabric is the PCIe or NVLink buffer. Both of these buffers are very deep, and waiting for them to be flushed can take up to 6 micro-seconds per hop.
>This is the reason that fences cause overhead when using the `Simple` protocol.

- **LL:** Designed for small messages, it can reduce latency significantly. Synchronization is done by passing sequence numbers with each piece of data (the documents call these `flags` which I do not like, they are sequence numbers in reality).
  LL uses 8 byte chunks, where 4 bytes are sequence numbers. This means that effective bandwidth is capped at half of the peak bandwidth and just wasted for metadata.
  LL is ideal for small messages on older hardware, especially if latency is critical. All it needs is an 8 byte write instruction to just put chunks on buffers.
- **LL128** is the bigger cousin of LL, where we chunk packets into 128 bytes and use 8 byte sequence numbers, increasing bandwidth efficiency. However, LL128 requires fancier hardware, since it requires large 128 byte atomic writes to move data into buffers which older hardware doesn't have.

### Communication Models

NCCL needs to handle both intra-node and inter-node communication models. The two operate on very different fabrics, and use different transports. Note that we distinguish between "protocol" and "transport" here in the sense that:

- A *protocol* is what was discussed in [[#Communication Protocols]]
- A *transport* refers to the packet-level protocol that sends and receives NCCL data units and manages flow control.

For each of the two scenarios (inter/intra-node), NCCL provides distinct suite of options for the users, tailored to the associated scenario.
We will discuss the details, but from a high level, the models are as follows:

![[Pasted image 20260411210748.png|500]]

We will now discuss how NCCL handles these two cases. We begin by laying out the properties of the communication fabric for each of the two cases, and then discuss how one may best make use of them.

#### Intra-Node Communication

NCCL employs a hierarchical approach for intra-node communication, with heavy emphasis on reduced latency. The bird's-eye view of the communication fabric is as follows:

![[Pasted image 20260412151324.png|500]]

The main workhorse behind intra-node communication is Nvidia's P2P transport, which heavily relies on NVLink/NVSwitch. 

>[!NOTE]- NVLink / NVSwitch
>As mentioned, one of the most costly things that we can do with NCCL in terms of communication, is doing `All-to-All`.
>Besides MoE, `All-to-All` also is needed when performing Tensor Parallelism (TP), where layers of a model are split across multiple GPUs, each one performing computation individually but sharing data globally.
>
>Usually, GPUs are just connected via PCIe, and while PCIe is great and a miracle of engineering, it is weighed down by its mission to remain _universal_, having to provide synchronization and fabric for a whole different range of technologies. PCIe gen. 5 for example is capped at around 128 GB/s of bandwidth, with latency fluctuating a bit depending how busy the bus is. These become bottleneck in a multi-GPU environment, where it is just impossible to reap the benefits of high compute over large data.
>
>To circumvent this, NVidia began designing *external bridges* for their GPUs that connect directly to dedicated ports on their GPUs. These are simple, low latency, low distance hops which are made almost entirely from copper. This is what we now refer to as NVLink.
>
>| ![[Pasted image 20260412221428.png]] | ![[Pasted image 20260412221516.png]] |
>|:----:|:----:|
>| NVLink bridges of different sizes | NVSwitch chips on a HGX motherboard |
>
>We won't discuss them much, but in case NVLink is not enough (e.g. physical placements of GPUs isn't allowing for point to point NVLink), NVidia manufactures motherboards with embedded NVSwitch chips that create full bisection bandwidth, non-blocking communication across GPUs in the same node.
>
>![[Pasted image 20260412222912.png|400]]

For fast intra-node communication, there are two things that we should strive for:
- **Little to no CPU involvement**: The mechanisms depend on the shape and size of data, but generally, sending/receiving small data repeatedly isn't well suited for CPU processing. Directly using NVLink or PCIe in lieu of that is much better. 
- **Minimal staging via system DRAM**: Any time we want to pass data along from GPU vRAM to system DRAM, it requires a `cudaMemcpy` call which is quite expensive.

NCCL dynamically adjusts the intra-node fabric based on available hardware. The best case is when `P2P_DIRECT` is available. This requires that communicating ranks be on the same process, allowing data to be passed directly via pointers to the same virtual memory regions (NCCL automatically handles synchronization, essentially implementing a small ring buffer that makes sure data is passed between GPUs correctly).

When `P2P_DIRECT` is not available, an intermediate FIFO buffer will be allocated in GPU vRAM that will be filled via IPC calls. Obviously, this will have significant overhead compared to `P2P_DIRECT`.

Despite all the above, NCCL may still use DRAM staging to move some amount of data in particular scenarios, or even use the NIC to do RDMA if the CPU bottleneck is too much.

#### Inter-Node Communication

NCCL threads are capable of direct interaction with NICs, and therefore, usually are the main driver for sending or receiving data between GPUs on different machines. NCCL also keeps a proxy thread on the CPU side which facilitates interaction between GPU and host memory and schedules data transfer operations through the NIC when needed.

![[Pasted image 20260422115809.png|500]]

NCCL provides two main methods of transports:

- **TCP:** Data is sent/received using plain TCP sockets. NCCL first pins some memory on the sender or receiver side as an intermediate buffer, the repeatedly copies data from the sender GPU memory buffer into it. Once the DRAM buffer is full, data is sent over the wire with typical TCP socket read and write.
  On the receiver side, the reverse happens, where the receiver fills the pinned DRAM buffer and then copies it into the host GPU.
  In order to initiate the transport, the sender and receiver execute a simple rendezvous protocol that ensures the pinned buffers are ready before TCP send/receive begins.
- **RDMA:** NCCL use the IB Verbs transport when capable hardware (either RoCE or InfiniBand) exists. How NCCL uses it though depends on one primary thing, whether or not the GPU and the NIC are connected to the PCIe switch:
	- **Separate PCIe Switches:** In this case, the general data-path remains the same as in the TCP case. Both machines pin memory in the host DRAM and then rendezvous with each other. The proxy thread on the sender/receiver sides then post an RDMA write/read operation and copy data from/to GPU memory and buffers.
	  The proxy thread no longer manages the transport, giving significantly better latency on small transfers and better throughput for larger ones.
	- **Shared PCIe Switch:** When the NIC and the GPU share the same PCIe switch, it is possible for the GPU to expose its memory lane to the NIC, meaning that all DRAM staging (i.e. the pinned buffers on the sender/receiver DRAM) can be skipped and the buffers on the GPUs can be filled directly.
	  Nvidia calls this method *GPUDirect RDMA (GDRDMA)*, and it completely frees the CPU side proxy threads as everything can just be handled with the DMA engines on the NICs.
