**Paper:** [Caching In the Sprite Network File System](https://dl.acm.org/doi/pdf/10.1145/35037.42183)
# Caching in Network File Systems

We have seen the benefits of caching multiple time, but one problem that we have not really discussed is *Where* should we cache?

This question is much more important from the client's perspective, since they have much more limited resources. In fact, so limited, that clients are better of not even caching!

![[Pasted image 20240214141614.png]]

Above table shows a fact that is even true now! **Accessing the memory of the server is faster than accessing the disk of a local machine! (Yes, even with SSDs!!!)**. 

Why? Because network delays are actually surprisingly low, so it is actually better to just let everything be handled from the server.

We have already seen write-through caching (i.e. once a client writes to local cache, update the server) for Coda file system. This has at least 2 big problems:
1. Client cache offers no benefit for write latency
2. Limits reduction in server/network load, as writes are still served directly and greedily

Client-side caching, beyond what mentioned above, is a promising approach for future, since:
1. Cache sizes are tied to memory size, which is rapidly increasing, making large caches feasible
2. CPU speeds are going up faster than any IO (both network and disk). Without a cache on the client, these machines become heavily bottlenecked 

One alternative is to use **Write-back Caching**, which updates the server asynchronously. So say each 5 seconds, apply writes to servers (defer work until you can't! Just like the previous lecture!).
- **Pros**:
	- Write followed by delete won't bother the server
	- It addresses both of the previous problems
		- Latency is improved, since clients can work with throughput of their local cache, and the server is bothered much less since writes can be grouped together in a single go!
		- Following from above, since group commits are possible, the IO is much less saturated on the server side!
- **Cons**:
	- Consistency issues
	- If client dies before updating the server, all writes are lost!!

There is not much we can do with the second issue, so let's discuss the first one. We have already seen this in the context of the Coda file system, and there, we solved it by invalidating the cache of all other clients when a client wrote to a file.

## Sprite Solution

Sprite solution is interesting. It goes like this, **If anyone opened a file for writing, force all clients to not use caches!**. What this means is that when a file is opened for write, all reads and writes for that file get sent directly to server.

Lets compare this to Coda:
- In Coda, you won't have to serve reads from server during writes as well, which is a potential benefit since reads are much more frequent than writes
- In Sprite, you won't have to burst an invalidation message to everyone that has the file opened each time someone writes something!

### Note About Sequential Write Sharing

There are at least two problems we need to address.

---
The first problem follows like this:
- A opens file, updates it locally, then closes it (note changes are not yet visible on server!)
- B opens file, populates cache with stale data from the server!

The solution is to force B to wait, and in order to do that, we take note of the fact that when A closes the file, the server will know it. It still may not have the updates, but it knows that A was the last writer to the file. Thus, when B tries to open the file, it blocks it and waits until the updates from A have propagated.

---
The second problem follows like this:
- A opens file, updates it locally and on server, then closes it
- B opens file, updates it locally and on server, then closes it
- A opens file again, but since it already has it in the cache, it will read it from cache, **which is stale!**

Note that the server won't invalidate caches in Sprite!. The solution is to associate version numbers with files:
- When file opened, a client will check if the version in local cache and server match
- If not, update the cache!

This is fast, since just checking a version number would just incur a network latency which is reasonably low.

---
## File Cache vs. Demand Paging

When a client accesses more files, cache size will grow, the only way to lower it is for other clients to write to it.

Since reads happen much more than writes, we need to be careful, since the memory is a precious resource that is used for many things on the client.

You cannot rely on a static partition of memory (i.e. say that the file system gets 4 GB of memory for cache), that yields terrible utilization, we need dynamic allocation.

And the best dynamic allocation strategy is LRU, as we remember for CS 402!
The problem? As we also remember from CS402, you can't do LRU in memory (at least the perfect LRU), since the perfect LRU would incur a *Page Fault* on very load/store and that is way way too slow.

LRU can be approximated though, with the LRU clock algorithm, which clears the referenced bit of a page after a period and then the PageOut Daemon will evict the resident page.

Sprite uses this same logic for caching. At each point of time, there will be 2 partitions of memory:
- One is the Sprite cache
- One is the virtual address space pages

Whenever either partition would have to evict, partition with oldest LRU entry will evict. 

## Evaluation

### Server Utilization

The file system was evaluated by running it through multiple benchmarks. Here, clients are *diskless* to make sure that nothing fancy can happen on the client side other than caching.

![[Pasted image 20240214154329.png]]

Two (unfortunately) familiar terms with different meaning appear here:
- **Cold** means that the server and client caches have been completely empty
- **Warm** means that test was first done on a Cold system, and then run again without cleaning the cache on either the server or the client(s)

The test was done with and without a client-side cache. For reference, the tasks here are:

| Task Name | Task Description                                                                                          |
| --------- | --------------------------------------------------------------------------------------------------------- |
| Andrew    | A benchmark made for the Andrew File System. Just a bunch of directory copies and a big compile and link. |
| Fs-make   | `make` the source code for the Sprite file system!                                                        |
| Simulator | Details don't matter, just know this thing does not do any writes!                                        |
| Sort      | Sort a 1 MB file                                                                                          |
| Diff      | Compare two identical 1 MB files, so it produces no writes!                                               |
| Nroff     | Do some text manipulation and formatting on the text of the original paper                                |

When no cache clients exist, things are generally the same, with one big difference with the Diff task. Again, this is taking the Diff of two identical files. Note that one of these files, lets call it file $A$ is on the server, and the other one, call it $B$ is on the client.

It is important to not that these tests do not have the same runtime, look at this table:

![[Pasted image 20240304030953.png|600]]

With a cold start, the client will make heavy use of the network, as it downloads $A$ from the server and do the diff locally. This download takes a long time, and indeed, the main different between the runtimes of cold or warm execution are because of that.

Now:
- Cache-less client:
	- Cold: Client asks the server for $A$. The server gives it to the client and the client performs the diff.
	- Hot: The server already has $A$ in its memory, so it immediately starts sending it. The big spike in utilization is because no disk IO was needed on the server side and thus thus jump in utilization is actually a good thing here!
- Client with ache:
	- Cold: Exactly the same as cache-less, cold client. After $A$ is fetched, the client keeps it in cache.
	- Hot: Client immediately sees that it has $A$ in its cache. It only checks the version on the server. Since it is the same, it just performs the Diff locally and we are finished without pretty much even touching the server!

### Scalability

![[Pasted image 20240214154713.png]]

Two things to note:
- This is most likely under-estimating, since it assumes all clients are active at all times
- Caches effect network utilization much more than server utilization, since reads are much more common than writes.

# How Are Things Now?

- Client-Side caching is the de-facto now, write-back caching not very common anymore.
- Sharing memory for file cache and virtual pages is still a concern
