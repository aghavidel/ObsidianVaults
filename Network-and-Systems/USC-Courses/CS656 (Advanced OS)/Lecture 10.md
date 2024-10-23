**Paper:** [Rethink The Sync](https://dl.acm.org/doi/pdf/10.1145/1394441.1394442)
# Improving Disk I/O Performance

First, let's discuss Synchronous vs Asynchronous I/O:
- **Synchronous**
	- Blocks until the request fully commits
	- Guarantees that request is fully committed when we return
	- It has poor performance, since it makes the caller wait, without being able to queue up another request
- **Asynchronous**
	- Spawn a new thread to commit the request and then callback
	- Gives much higher throughput, as much as the thread scheduler can (which is much more than disk IO speed usually)
	- It foregoes the clean guarantee that we had for synchronous requests. It is no longer obvious that the IO state (e.g. the disk) will correctly reflect the served requests. The only way to make it look like that is to wait for IO completion, which is essentially just the synchronous model.
	- There is also the concern that the scheduler can serve requests out of order, unless requests are bundled which is more complex.

For the purpose of comparison, EXT3 is the file system that we use as a baseline, and it turns out it has a weird "feature":

>[!NOTE] Note About EXT3
>EXT3 returns when commit is done on the *disk controller buffer*, NOT the disk itself. This means that even in synchronous mode, there is no guarantee that the data is on disk when we return!

## External Synchrony 
The main thing here is that we only need to make sure that the write is done *when we say it is done*, so just hide everything from the user until we are sure. If the user is not able to tell the write has been applied, then there is no reason to make sure it is done by that time.

The implication of this is that we need only commit a disk IO action, when that update is actually becoming externally visible. 
This means that we would be effectively indistinguishable from a synchronous file system, but since we are not actually blocking the kernel, we can bundle disk IO actions and essentially have the performance of the asynchronous system.

![[Pasted image 20240304002100.png|500]]


>[!FAQ] External Output
>The notion of "External Output" is anything that changes the *stable* image of an operating system. What that includes are things like:
>- Disk commits
>- Output screen writes
>
>And many more. These are things that *can* persist across an OS crash, and since we cannot predict crashes, we need to be careful with them.

This deferral of disk IO actions are beneficial since:
- Many updates can be batched together
- Large sequential writes can utilize the disk better
- If an update is immediately overwritten, we can just skip it

The only question here is, how **much** can we defer it?
The best policy would be to defer it, until another IO update comes in that depends on that update being complete. 
For example, if we commit data to disk and then attempt to print it on the screen, that printing action is an IO call that depends on that data being on disk, and as such, when that call is made, the disk commit can no longer be deferred and must be immediately scheduled.
Also, if the disk cache for all of these operations is becoming full, we may also decide that we are not able to defer any more calls and need to commit.

## Evaluation
The main evaluation would be on how much operation throughput we would have..

![[Pasted image 20240212152517.png]]

This is the easier one to process:
- We only suffer about 7 percent compared to the asynchronous EXT3
- We can already see how high the penalty for the synchronous file system is when it comes to disk I/O

The more interesting test is the test where we untar the whole Apache web server source code.

![[Pasted image 20240212153222.png]]

Two things to note:
- The benefit from `xsyncfs` compared to EXT3 with synchronous output is much smaller, since the untar process has a lot of output, so disk commits cannot be deferred much.
- There is also quite a bit of computation involved, and this is why RMAFS is also used as a benchmark. It shows a lower bound on how much that process actually takes, and so the part of the graph that comes from disk IO actions should be found by subtracting this value from all the graphs. Doing so, gives essentially the previous graph.

This is much more representative of how a files system would be actually used compared to the previous one.

We also have a web based tests.

![[Pasted image 20240212153745.png]]

This is interesting, since `xsyncfs` is actually performing *worse* here. The reason is that in a web based scenario, once we receive the request, we cannot delay, since the client is actively waiting for a response. So there is not much time to defer anyway and thus the extra baggage of `xsyncfs` actually hurts it here, since it is essentially performing fully synchronously anyway!

In general:

![[Pasted image 20240212154133.png]]

As you can see, the gap closes as the IO size gets larger. This makes sense, since the commit takes so long for large requests that it dwarfs any amount of queuing that we do on the disk cache.
For small writes we have 33 ms delays, but that is OK, since such a delay for small writes is already invisible to the human eye!