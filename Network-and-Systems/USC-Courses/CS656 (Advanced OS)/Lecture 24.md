# Resource Disaggregation

Until now, we saw a cluster or a data center as a series of individual nodes, however, as the technology has progressed, the view is starting to change. The boundaries between nodes is getting blurred and it is becoming important for nodes to cooperate and access the memory on each other.

To this end, the mental model of a cluster is moving into the resource pool model, where we have hardware and software that runs on different machines, but looks very homogenous.

![[Pasted image 20240415122048.png]]

This model is pretty new, and the main reason that it is now possible, is that network is much MUCH faster now. Round-trip time in data centers is orders of magnitudes less than average HDD seek, taking as few as tens of micro-seconds tops, and moving into a few micro-seconds.

One of the main things that powers this huge speedup is Remote Direct Memory Access ([[RDMA-Intro]]).

![[Pasted image 20240415122303.png|300]]

The goal is to bypass the kernel on the sender and the CPU on the receiver.

## Infiniswap Intro.

Infiniswap is an attempt to implement distributed memory through disaggregation, i.e. creating a data center wide memory. This is pretty simple with RDMA, but the main thing to not is page eviction. You cannot just evict a page locally, it needs to be evicted in the remote memory as well.

Infiniswap works based on 3 principles:
1. Performance impact of partial working set being in memory is non-linear. This means that even small improvements of being able to keep the working set in memory will heavily improve performance. Thus, remote memory can dramatically speed up the performance.
2. There is significant skew in memory usage. There is a lot of spare memory.
3. Spare memory has some temporal stability, which effectively means that spare remote memory is not immediately evicted.

![[Pasted image 20240415124808.png|500]]

There are challenges:
1. How to makes this backwards compatible?
2. What if spare memory goes away?
3. How to deal with machine failures?
4. How to even find a machine with spare memory?

## Implementation

The goal is to ideally, evict to a machine that has the most spare memory. This minimizes the likelihood that the memory will go away.

But how to find spare memory?
One way would be to rely on a centralized coordinator, that of course is a bad idea if it explodes. It is important to note that we don't need to be optimal, as long as the remote machine has more than some small threshold of spare memory, then that would be enough. 

![[Pasted image 20240415125809.png]]

We rely on a pretty old technique, choose 2 random machines and pick the one with the least loaded memory. 
Once we know who has spare memory, we should decide *what* to evict. Usually we would just go for another NRU method, but that is very hard here. Why? Well we don't have the kernel to help us since RDMA bypasses the kernel! Local machines won't even know remote access had happened to them!

So we again opt for an approximation, we evict only lightly loaded pages.

To cope with failures, we must keep the two copies (local and remote) consistent, so any write that we do on the remote memory should be done on the local disk as well. The problem now becomes, when the eviction is actually complete? Should we wait for both writes to finish?
No! One of them finishing is sufficient. It is easy to detect failures since memory access will fail if a machine dies. If a remote machine dies when a local disk operation is not done, then we need to hope that the local disk is consistent with the order of operations.

To tackle backward compatibility, we need to make Infiniswap transparent, and to do that, we need only sit below the VMM.

![[Pasted image 20240415130727.png|300]]

This means that anything that runs in the VM needs not care about whether or now Infiniswap is used or not.