# 1RMA

**Paper:** [Re-envisioning Remote Memory Access for Multi-Tenant Datacenters](https://dl.acm.org/doi/10.1145/3387514.3405897)

# Background: RDMA

We have already discussed a bit about RDMA before, so let's not bother too much with it. 

![[Pasted image 20240424140823.png|300]]

In general, for an RDMA operation:
1. Install the `verbs` API via the `libibverbs` library
2. Register operations to the driver in kernel which create *Work Queue Elements* for the NIC
3. Pass the memory buffer address directly to the NIC and let it do stuff to it without kernel involvement 

We have in general, two types of operations:
1. **One-Sided** operations that do not need the help of the remote CPU. These include read and write operations, where we only register the local and remote buffer addresses to the local NIC, and then the NIC will read from the remote NIC and put it in local or write the content of our local buffer to the remote buffer
2. **Two-Sided** operations that need the remote CPU for some data manipulation. These are send and receive operations. These are essentially the synchronous version of the above operations. The two sides agree when to send/receive the data, instead of the data just materializing in the local or remote buffer.

>[!EXAMPLE] RDMA `Send`
> To do an RDMA send, let's keep the following picture in mind:
> 
> ![[Pasted image 20240424141541.png]]
> 
> Here are the steps:
> 1. The applications posts the send operation into the send queue of the local NIC
> 2. The local NIC dequeues the operation, sees the registered local and remote buffers and then DMAs the data to the remote buffer.
> 3. The remote NIC, sees that it has an operation sitting in the receive queue, it reads it and sees that it was a send operation. It then retrieves the data that was just DMA'd into its memory and posts a work queue element into the **Completion Queue of the Local NIC**
> 4. The local NIC, during this process, polls the completion queue to see if the remote did receive and process the packet. Upon seeing the element, it notifies the application of the send operation completion.
> 
> This is how this operation is synchronized compared to a normal write operation. 
> 
> Pay attention to how packets can get *reordered* along the way. The NIC should be able to handle that!

As you can see, all of this sending and receiving packets lack congestion control from the kernel. Indeed, RDMA implements its own congestion control. The one that we see most is **RDMA over Converged Ethernet (RoCE)**, which is a *backpressure* algorithm, it prevents the senders from sending more packets if it notices that the queues have exceeded a certain threshold. 

RDMA is very performant, and it gives an impressively tight guarantee on latency that no kernel implementation can match, both the median and p99 of latencies can be within a few micro-seconds on a healthy network.
However, there are problems. In particular:
- **NIC Memory Is Small**, since the NIC cannot rely on the kernel to keep connection state for it, it needs to keep it in its own memory, and can exhaust the memory limit.
- **High Priority Applications Can Get Starved**, this is due to head-of-line blocking. Write operations in particular impose a FIFO ordering of queue elements and as such, if an operation is delayed, everything behind it is delayed as well.
  This is a classic **Priority Inversion** problem, where a low priority but slow flow on the head of the line can block the completion of a high priority flow behind it.

There is also a host of other things. One scary problem is that RoCE, which relies on backpressure for creating a loss-less network, can actually cause a deadlock in the data center if there is a cyclic dependency on the communicating components (imagine a ring of nodes, with slow links, screaming at their successors, the queue fills up in the same way for all links, so RoCE will pause all nodes at the same time, and hence all nodes for stop sending, **AND** receiving at the same time, the queues will never drain and we now have a deadlock!).

There is also security issues. The memory regions that are registered need to be protected, and that requires encryption. Large keys cannot be used, since that is just too slow, so smaller keys are used that are repeatedly changed. Re-keying a connection is possible but still rather slow.

This paper proposes a different way of doing RDMA, the key ideas are:
1. Forget two-sided operations, just support one-sided ones
2. Use the CPU to implement better congestion control
3. Simplify the NIC and make it connection-less!

## 1RMA Operation

The basic operation is as follows:

![[Pasted image 20240424143206.png]]

1. The local NIC reaches out to the remote NIC. It authenticates and asks for the remote NIC key
2. The remote NIC gives the local NIC its key and then the two are ready to start a secure transfer
3. The local application enqueues its operation onto the NIC, here it is a `read` operation
4. The local NIC, sends the op packet to the remote NIC
5. The remote NIC reads all requested memory regions from memory and sends them
6. The local NIC receives all responses and will then write them to memory
7. When all operations are finished, the local NIC signals completion to the application

**All of this communication is encrypted!**

##  Memory Region Encryption

Encryption is necessary here. Clouds host 3rd party applications and as such cannot trust them. They are particularly worried about:
- Accessing unauthorized memory regions
- A hacker injecting packets and observing traffic

So how to make this secure?

![[Pasted image 20240424143916.png|500]]

Encryption is very much possible only with hardware support. [[AES-GCM]] is the technology that powers it.

Memory is partitioned into memory regions on both sides. Each memory region will be assigned some region key $K_r$ which only the local NIC knows. From this key, a series of *derived keys* can be generated ($K_d$) that can be used to communicate with a remote region. **Neither of these two keys are ever sent over the network**. Each key is 128 bytes and fits comfortably in the NIC memory, **but only the first key actually needs to be in memory!**

Above you can see an example:
1. A process with a $PID_{initiator}$ and bound to a NIC with $ADDR_{initiator}$ wants to perform some operation with a given `OpType`. It sends all of this data with a secure encrypted RPC (i.e. it can be slow to initiate this).
2. The server receives that data, and by reading it, generates a key $K_d$ for this communication as:
$$
K_d = AES(\text{Key} = K_r, \text{Contents} = (ADDR_{initiator}, PID_{initiator}, OpType))
$$
3. The server sends back the derived key for the region that is available to the local NIC
4. The local NIC then uses that key to sign each operation request, and it also includes both its address and its PID on each request.
5. The server receives each packet. As mentioned, the $K_d$ isn't actually in memory, so the server actually regenerates the key using the header in the operation request and then checks to see if the signed content is valid. If it is, the operation is served.

None of this can be done if verifying $K_d$ for each packet could not be done at line rate. It is also important to note that this still won't solve if the original host itself is compromised, and as such, key rotation is required to be done repeatedly and to force re-authentication with the secure encryption protocol.

