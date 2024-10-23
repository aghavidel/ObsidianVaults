**Paper:** [Difference Engine](https://www.usenix.org/legacy/event/osdi08/tech/full_papers/gupta/gupta.pdf)
# Intro.

Even back when this paper was written, VMWare was already a pretty mature technology.
The main reason for why VMs became so popular, had to do at least partially with higher utilization.

Of course, if you have 32 cores, you want your VMs to make use of those cores as best as they can. The problem however is that each VM by itself tends to be underutilized, really only using the resources about 5-10 percent of the time. 
So you need much more than 32 VMs to utilize them correctly (unless you have very CPU-intensive processes).
The problem is that fitting many VMs under a single machine, means that the machine needs to provide a huge amount of memory. Multiplexing this virtual memory to these VMs in an efficient manner is quite the challenge. 

So, how to cope with that?

1. **Add more RAM!**
	1. Problem with hardware can arise. Utilizing more memory correctly, requires better hardware in general.
	2. More power and more money is needed.
2. **Swap to disk!**
	1. High overhead, if the working-set is not in memory, which can happen much more easily if memory is under contention.
3. **Use less physical memory, use stronger abstraction!**
	1. The minimum physical memory is the sum of the working sets, not the sum of demands.

# Other Approaches

There are 3 main approaches to tackling this problem:
1. **Page Sharing:** If multiple clients are using the **exact same page**, then share it among them
2. **Page Patching:** If clients access *very similar pages*, then the we can keep one and then store only the diff of the other one.
3. **Page Compression:** Compressing the page content as a whole.

![[Pasted image 20240318142114.png]]

How to share pages?
This is a classic problem which we know the solution of from CS 402, that being Copy-on-Write. Once you share a page, mark it as read-only, once someone tries to write to it, we get a page fault and to handle that, we create a new private page and fix the page table.

The only modification to COW needed is a reference count, since there can be multiple people pointing to the same page, so we keep a ref-count and each time we copy, we decrement the page. The moment it reaches 1, we know the page does not need to be read-only any more.

>[!REMINDER] Xen And Paravirtualization
>Xen, unlike VMWare, uses paravirtualization, where the guest OS needs to run a Xen-aware device driver(s) to actually work with Xen, so the guest really *knows* that it is running as a virtual machine.
>
>The main benefit of this compared to VMWare with its full virtualization is a performance boost, since instruction conversion is not directly needed like with VMWare. One specific challenge of COW in paravirtualization is that in order to prevent circular dependencies, we should never share pages with the domain-0 VM. 
>The reason is that the page fault handler itself needs memory to execute code, and it lives in the domain-0 VM, so if we share pages with that code, then the page fault handler itself can get a page fault!!

There are a couple things to note:
- Patched/Compressed pages have high access overhead, so we need to be careful with not making too many of them.
- We should also note that referenced pages should *NOT* be used as references for other page patches or compressions.

##  When to Patch/Compress

Difference Engine implements a Not-Recently-Used (NRU) policy which is more fine grained. There is a dirt bit and was-read bit. 

So we can add a referenced bit `R` and a dirty bit `D` for modifications, and we can have the following:
- If `R = D = 1`, then no page sharing on this, for all we know, it is completely unique data
- If `R = 1` but `D = 0`, then this page is a good candidate for sharing, since it is not being modified. It can also be a reference for patching.
- If `R = D = 0`, then the page is a candidate both for sharing and also for patching.
- If `R = D = 0` for a long while, then it is a candidate for compressions or being swapped to disk.

## Evaluation

![[Pasted image 20240415053548.png|500]]

First, let us look into the overhead of each of our operations:
- Page sharing, COW, compression and un-compression are pretty cheap
- Page patching is expensive, since all the hashing and choosing the references takes time, yet un-patching is cheap, which means that as long as we honor the NRU algorithm, we are fine.
- Swap-out is fine, as long as it is done asynchronously, but swap in is very expensive since disks are slow and we MUST wait for it.
