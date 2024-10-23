**Paper:** [LFS](https://dl.acm.org/doi/pdf/10.1145/146941.146943)
# Log Structured File Systems

We have already discussed RAID, but it is not enough yet!
1. RAID still will have a high cost
2. Utilization of **each disk** is still terrible, having an array of them just uses statistical multiplexing to increase performance. For reference, utilization of each disk in RAID stands around **5-10 percent!**

The main bottleneck of each disk is always the *seek time*, the time to actually get to the data itself, whether it is a write or a read, the lower bound of the seek time is always high since it is a mechanical action.

The solution? Try to eliminate the seek time all together. We use a log for the **whole** file system. This is in contrast to the log that we discussed for the System R paper. Here:
- The whole data *is* the log, the point is that writes are equal to appends to the log
- Similar to it, we don't do updates in-place, appends won't overwrite the previous data immediately!

This creature is the extension to the ***Sprite File System*** (keep that name in mind, we'll revisit it soon enough), and is called the Log File System or LFS.

>[!FAQ] Comparing With System-R and Shadowing
>In System R, the main reason that we used a log was so that we would be able to keep the transactions and repeat them when clients reconnect. The main difference here is that unlike system-R, there is no separate datastore for this, as we mentioned, the log *is* the data itself.
>
>Shadowing is also a similar concept, as it essentially makes use of the same principal, *don't do updates in-place if they might have to be rolled back, do it in a separate memory space.*

So it is all good for writes, so what about reads?
For reads, just cache the whole thing! We have fairly large memories now, so just keep everything in memory as much as you can to minimize disk I/O.

>[!EXAMPLE] Updating Data in LFS
>Let's assume we make a change to the file in `/home/harsha/555/notes`.
>To make this change, we write a new block to the log:
>
>![[Pasted image 20240303164850.png]]
>
>Done? No!
>The problem is that the I-node for `notes` is not pointing to this new block, thus if some other user tried to access this file, they won't see the new updates. Thus, we need to update the I-node as well. Assume we have some construct that let's us do that for now. Just note that this too is a *write* operation, so the new I-node *also* goes on the log!
>
>![[Pasted image 20240303165107.png]]
>
>Now what? Are we done? No!
>The problem is that now, the I-node for `555` that is the directory that holds the `notes` file is pointing to the old block, we need to update that one too! And on-and-on it goes until we reach `root`.
>
>![[Pasted image 20240303165222.png]]
## Challenges

Looking at the example we just gave, we can see at least two fronts of trouble brewing:
- How to find the ***latest version*** of a piece of data (e.g. *someone* should know that we made a new I-node for `notes` and all of its ancestors, who is that?) ?
- How to ***cleanup*** the disk from all the garbage we make (e.g. all the old I-nodes in the previous example)?

### Finding The Data

Use a new data structure, an **inode map**, a data structure that maps inode number to its disk block, and lives in the memory. The authors note that disk access is rarely necessary to access these maps. The reason is that:
- Memory is large, so we can cache quite a lot of data, including more maps, so it works well
- We can also exploit temporal locality, since recently used files will use the same I-node map, so it makes sense that we use them much more than a random I-node map.

The I-node map is kept in blocks that are then written to the log when a change happens. 
In typical Unix file system, a fixed portion of a disk (usually near the beginning) is allocated to I-nodes, but in LFS, this equivalent portion is used only for *checkpointing*, keeping a reference to all written I-node blocks and periodically updated.

For comparison with Unix FFS, here is a typical view of the disk segments:

![[Pasted image 20240303181357.png]]

In case of a system crash, it is possible that the data in that preamble is not up to date, and as such, we are forced to scan the whole disk. LFS accepts this cost, it is not designed to minimize crash startup time, it is designed to minimize anything that comes after it!
### Garbage Collection

The main attraction of LFS is that writes are very fast, but that will only be the case if the head of the log only faces a large, contiguous, free block that can be used to write anything. As such, if the log gets dirty because of all the garbage such that this block is no longer free or just too small, the performance of LFS will degrade significantly.

Faced with this, it is inevitable that we might need to do some *defragmentation* by moving the blocks, which means that we should copy them into some other place (e.g. memory, another log or another file system) and then compact them together, then write them back on the log head.

This process however takes way too long to be feasible on a whole disk, thus, we need to divide and conquer.

We divide the log into *segments*:
- For writes, we just choose a clean segment and write sequentially 
- For garbage collection, we just read in a full segment and copy the *live* data to the end of the log and write back the rest.

The problem is that as the segments get more utilized, the overhead of copying to the end of the log becomes larger and we get worse performance. To reason about the performance, we introduce the notion of *hot* and *cold* segments.

A hot segment is a segment that has been recently updated with live data and a cold segment is one that has not been touched for some time. The key to making sure that the file system is performant, is to increase the amount of cold segments as much as possible. Basically, we need to be less greedy with cleaning up "warm" segments, since if we leave them alone for some time, there is a chance that they become cold enough.

For example, if we use a very greedy cleaning strategy by just cleaning hot segments (note that hot segments will contain quite a bit of garbage since they have been overwritten a lot), we get very bad performance. The reason is that even if we do clean them, they are going to become dirty pretty soon!

So we should try to be less greedy and leave the hot segments alone for some time and only cleanup the cold segments. So we need to use some prediction here, since the benefit scales with how long a block of data will stay untouched. 
When designing page tables, we also have the same problem, so we can use the same LRU strategy here as well, just garbage collect the coldest segments and keep going until we think it is enough.

#### Segment Cleaning Implementation

It is worth going into the implementation a bit.
Each segment will contain at least one *segment summary block*, which contains a summary of the segment contents. For example, for data blocks, it will contain the file number and block number for the data block.

>[!FAQ]- Why Multiple Summary Blocks?
>If more than one write is required to fill a segment, multiple summary blocks will also be needed to fill it. 

The more important content of the block is also whether or not a piece of data is *live*. Once we know the identity of a block, we can determine whether or not it is live or dead by looking into its corresponding I-node and then hopping to the associated summary block.
If the summary block does not point to the block under question, then it is dead, else it is alive. 

This is essentially the same coarse-grained LRU implementation that we used for memory pages, where we used a `referenced` bit to do the same thing. Sprite however, performs a little optimization as well, it keeps a version number in the I-node map entry for each file which is incremented each time a file is deleted or truncated to zero as part of updating the I-node map. This is also stored in the segment summary block.

Now, if we inspect a block, look into its version in the summary and it does *not* match the version in the I-node map, then there is no need to jump to the block I-node itself (which can save a seek time), we can clean it immediately.

So we now have our LRU implementation. We use it, and lo and behold, **it's bad, it's explosively apocalyptically bad**!
#### Comparing Cleaning Policies

Now that our simple LRU has failed miserably, we need to look into other policies, and for that, we need to be a bit more concrete.
The paper introduces the notion of *write cost*, defined as **the average amount of time that a disk is busy, per byte of new written data**. This definition includes all disk accesses, even for cleaning (duh!).

For context, if write cost is 1, then that is ideal! It means that all the time that we are accessing the disk, it is purely for the actual writing of the bits. On the other hand, if it is say, 10, then that means that only one tenth of the time of the operation was actually used for writing the new data, the rest was spent on:
- Cleanups
- Seek times
- Writing other data structures

And everything else. 
With LFS, assuming it's implemented correctly, seek and rotational times are negligible, so we need only really worry about the cleanup time, as the time to write the other data structures is also negligible. 
This can be expressed in terms of the disk *utility*, $u$, defined as the ratio of live segments in the whole disk. This means that if we read $N$ blocks, $N.u$ of them need to be written out and the rest must be cleaned.

Now, the amount of data written to disk is variable, but in a steady state, where the disk is in full saturation (e.g. the worst case in some sense), the disk accepts writes just as equal to the amount that it can actually free. In simpler terms, once we clean $N$ segments and append the cleaned parts to the head of log, all of that is consumed to serve the write and this continues. With a utilization of $u$, this amount is of course $N.(1-u)$.

So:
$$
\begin{aligned}
	\text{Total Read Segments} &= N\\
	\text{Total New Bytes That Can Be Written} &= N.(1-u) \\
	\text{Total Write Backed Segments} &= N.u
\end{aligned}
$$
The total write cost now would be:
$$
\text{write cost} = \frac{N + N.(1 - u) + N.u}{N.(1 - u)} = \frac{2}{1 - u}
$$
This is true, with the assumption that **a whole segment MUST be read as a whole to determine its live segments**. This is conservative; for example, if the read segment has no live segments (i.e. its utilization is 0), then we can just write it on the log head immediately, giving us a write cost of 1!

Here is a graph and comparison with FFS:

![[Pasted image 20240303195702.png|600]]

The graph above shows the write cost. Note that the value for $u=0$ is actually 1, but it jumps to about 2 if $u > 0$ as we discussed above, and only grows with increasing $u$.
This shows that LFS works best when the cleaning procedure is not needed too much, which is equivalent to saying that the disk utilization is actually pretty low. This means that there is cost-performance trade off for LFS (note that "cost" here means money, not performance!).

- If utilization is low, you essentially have a smaller disk for the same price, but it runs faster!
- If utilization is high, you will have your money's worth of disk space, at a big performance penalty.

Since disks have been prioritized to be cheaper, and LFS is actually quite complicated, it was **never actually deployed widely for HDD disks. It's a different story with SSDs though!**
With this though, the writers still tried to pull through and make LFS cost-effective. Note that the analysis above was done on a steady-state assumption, which is not true. The thing is that writes are not that common, and thus cleaning up only for it to get immediately consumed is most likely indicating of a bad implementation, rather than heavy utilization!

With this in mind, the best way to use LFS in a cost-effective manner, is to try to operate on what the authors call a **bimodal distribution of utilization:**
- The majority of the disk segments are fully utilized (makes it cost-effective)
- There are a few segments that are empty or very nearly empty that the cleaner can use

This is a hypothesis that needs to be tested, so lets move on to the evaluation.
## Evaluation

### Cleanup Overhead

The authors created a simulator that runs a file system in two modes (here, an "access" means opening and rewriting the file content)
- **Uniform:** All files are accessed with equal probability
- **Hot-Cold:** 10 percent of the files are "hot", they are accessed 90 percent of the time. The rest are "cold", they are accessed only 10 percent of the time.

Note that Hot-Cold is much more realistic than random.

For evaluation we need to see:
- How do we behave under different workloads 
- Do we actually utilize the disk better?

![[Pasted image 20240303201802.png|500]]

The result of this graph is surprising. <u>The hot-cold scenario runs <b>worse!</b></u> This goes back to the LRU implementation that we mentioned before. The problem is that it is **greedy**.

With hot-cold simulation, the cleaner will gravitate towards cleaning only hot segments, but the problem is that **they are going to get dirty soon anyway**! This isn't a problem on an LRU implementation for page tables, since DMAs are very fast, here though, it creates a big problem.

The solution is to treat hot and cold segments differently. Hot segments are well, hot, and we are better off not touching them until they cool down. Since hot segments are rare, and so the cleaner ignoring them would not cause too much disk underutilization. 
For cold segments, the story is different, these are the ones that the cleaner should be the most interested in, since if we clean them, it is going to take more time until they are accessed again by the file system.

To formalize this, we can use a cost-benefit notion. The cost of cleaning a block with utilization $u$ is $1 + u$, which is the cost of reading it and writing back the live segments. The benefit is something that we can define, and we define it here by $(1 - u) \times \text{age}$. 
- $(1 - u)$ is the amount of data that we will free. If something is going to give very little data, it is not worth cleaning it.
- The age here, is essentially the distance from the timestamp of when the segment was last modified. We can afford to keep this timestamp since segments are quite large (in contrast to say, pages of memory!).

So the benefit over cost would be $\frac{(1 - u) . \text{age}}{1 + u}$, and we clean the segment that has the largest benefit over cost, and we call this implementation of LFS the "Cost-Benefit LFS".

How does this fair?

![[Pasted image 20240303203413.png|500]]

Lets parse this figure:
- The y axis is showing the fraction of segment with a particular utilization. A greedy LFS would clean a segment with more than a particular utilization value, but that has the problem that over time, the number of highly utilized segments get larger and larger!
- For Cost-Benefit LFS, we actually approximate the "Bimodal" distribution that we hoped for! There are still high utilization segments, but there are also a lot of low utilization ones that we can use for cleaning!

So now that we are operating closer to the Bimodal distribution, the overall performance should have improved:

![[Pasted image 20240303203731.png|500]]

As for how this is actually implemented:
- The last modification time, is the timestamp of the modification to the youngest block of a segment and is kept in the segment summary block.
- A new block, the **Segment Usage Table** is also introduced, which keeps track of the live bytes in a segment. A segment with no live bytes can be used without cleanup. This data is modified upon any update to the segment and is stored in the checkpoint region of LFS.

### Performance

![[Pasted image 20240303204525.png|500]]

Here, they created 10000 one-kilobyte files, read them and then delete them and measured the throughput by number of files per second passed. LFS is evaluated against SunOS, and in figure (a), it is done on the same CPU always, but with (b), file creation is done with more CPUs added.

The key insight is that:
- LFS is MUCH better for writing, and thus has a huge performance boost for creating or deleting a file, and all of the optimizations are also good for reading.
- With SunOS, having more CPUs is NOT helping the IO, but LFS is improving. The reason is that since the disk is much busier with SunOS, CPU cycles get wasted waiting for disk IO to finish up, but in LFS, throughput scales with CPU cycles, since the disk is not being a bottleneck!

One thing though we need to discuss, and that is the performance for re-reading the data with large files.

![[Pasted image 20240303205116.png|500]]

Since they are large, they won't fully fit in the cache, and as such, we get cache misses after some point and need to seek the data.
The problem is that:
- For LFS the whole data can be spread between multiple cylinders, so a seek might be needed.
- For the SunOS, they can optimize by putting things on the same cylinder and not have to do much seek.

With this in mind:
- Sequential writes are faster with LFS, since writes are much faster in general, so random writes are also very fast (in fact much faster, since SunOS suffers a lot of unnecessary seek times)
- Sequential read has only slight performance drop, because we have an extra I-node map lookup
- Random read strains both file systems, since they have to seek on the disk regardless
- Sequential read is probably where LFS gets a big hit, since unlike SunOS, LFS has actually spread the data around and has to seek on the disk to catch them!

### Real Workloads

Cleanup has less impact in the real world than simulations predicted, because the "cold" data is even cooler than they expected! Also, large files tend to be created/deleted in their entirety.

Also, the cleanup overhead can be further decreased by deferring cleanup to when the disk is most idle, for example when there is not much activity going on (at night for example).
## On SSDs

LFS is actually really good for SSDs!
Why?
- Random reads are very cheap on SSDs, so we don't really need caching any more!
- For overwrite, SSDs need to erase large chunks by default, so LFS log cleaning is a good fit.
- The log policy actually spreads outs the writes to different portions of the SSD, and that is good for the SSD health since it won't be re-written repeatedly (SSDs are bad with repeated re-writes, they will wear out)

Ironically, LFS is now pretty much only used on SSDs, despite the fact that it was created with HDDs in mind! The Flash Translation Layer that controls the SSD is pretty much LFS!