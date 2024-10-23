**Paper:** [Disconnected Operation on Coda File System](https://dl.acm.org/doi/pdf/10.1145/146941.146942)

The paper considers the design of a "*disconnected*" mode of operation for a file system (Coda FS). Here, disconnected means that there is no link to the shared repositoy of files.
The working scenario assumes there are many untrsuted clients, and only a handful of trusted repositoies. When a client goes down, they lose access to the trusted sources and as such, they won't be able to continue working on their files.

To help metigate this, the paper proposes that we use **caching** on the client side. This is already used to improve the performance on the server side, but not used very much on the client side. This use-case however gives it a concrete and well-defined purpose. 

# Intro

This paper was among the first to really seriously discuss File System as A Service (i.e. ancient Google Drive).
There are many good reasons why we want that:
- Larger capacity
- Fault tolerance
- Portability across different OSes and Hardware
- **Collaboration and Sharing!**

The downside is that:
- It is more complex
- ***What to do if you don't have an internet connection?***
## Caching

Caching was already used, a lot on server side, much less on client side. In particular, the most popular client side caching strategy is **write-through caching**.

Write-Trough caching means that:
- When there is a new read, cache the data on the client side and serve all reads on the client
- When there is a write, update local copy and then push it into the server

Looks simple! It lowers read latency and keeps the server less busy, allowing it to scale much more. However, there are problems:
- **Data Consistency Issue:** If multiple peaople write to their caches, if they are doing it the same file, then pushing carelessly to the server can overwrite their changes.
- **Cache Size!** The user technically needs unlimited cahce. You need some cache eviction policy
- **Availability** If a user keeps writing to a file, every single request should be propogated. That is not efficient.

This can be helped somewhat with **Write-Back Caching**, batch writes together for some time, and then apply them in one fell swoop. But still, the bigger problem is a problem with all caching solutions, **Conflicts!**

"INSERT FIGURE OF PAGE 5"

Conflicts happen when someone updates the file but it does not propagate to the other users. If people write to the same portion, the server should deny one of them and send an error to them to retry. 
When there is an update, the server should update not only the server copy, but every cached copy as well. This means that the server should be able to invalidate some client caches if needed. This prevents conflicts, but requires optimizations. If the server were to send a full updated copy for each write, the bandwidth usage will scale with file sizes, and that's really bad.

The better solution would be to invalidate the cache instead, send a message that causes the client to **miss** when they access that cache entry. This message would be small compared to the data sent, and as such, scales much better without eating bandwidth.

>[!IDEA] Caching For Availabiloty
>When a client forcfully disconnectes, keep the local cache and server requests from there. When they come back, reconconcile the cache with the server.


## Challanges

Caching for availbility has challanges:
- What if the file isn't in the cache?
- How would the reconciliation on the server work?
- How to handle conflicts (of which there can be many, since user can be disconnected for a very long time)

If the user tries to write to a file that hasn't been accessed for a long time, then there will be a cache miss and then there is nothing we can do.
There are also 2 other things, those being:
- If the user has a large cache, then at some point they might evict an entry out of concern for space.
- If the user is working on a very large file, they may not even be able to fit it in a cache in the first place!

The solution here would be to break down a file into smaller blocks and cache the blocks instead of a file. This could increase the miss rate for large files but at least makes sure we could cache *something* even for very large files.

The problem now becomes, how do we increase our cache hits? There are some insights:
- Users have preferences among their files. Some files are really important and some not. We can let them flag those files for us so we can ache them ore aggrasivly.
- Caches usually exhibit strong **Temporal Locality**, a recently accessed file is most likely going to be accessed again.
- Cache at the level of the file, cache miss when a file is opened (**Didn't get this, look into it!**)

# Cache Hoarding

We want to cache files aggrasively to prepare for disconnects. To make this feasible, we can give priorities to them. These priorities would decay over time when cache is not accessed and are rvitalized when accesed. 
We can then priodically fetch high prority files which are not already in cache.

## Cache Reconciliation

One naive way to reconcile after diconnect, would be to sent the whole cache.
This has a high overhead and more importantly:
- Only part of the cache may have changed since disconnect
- Not the whole file might have been changed in the cache

The solution, **maintain a log of updates during disconnect**. We send the log after reconnect and replay it on in the server to upate the files.
There are many optimizations you can do to reduce the size of the log as well. We don't need to discuss them at length.

## Preventing/Identifying Conflicts

Conflicts cannot be prevented in a distributed system like this, unless very aggreasive locking mechanisms are used. The server also might have to deny access to files that could be modified by disconnectd users to make sure this never happens, so it is not efficient.

It is much better to patch things up when they break instead of making sure they never break in this situation. So **detecting** conflicts is much more important.

"INSERT FIGURE FROM PAGE 13"

Put a version number for each file, upon update, note what the version of the updated file and the version it is updating is. Conflict when the version of the client is not equal to the version of the server plus 1.

# Evaluation

What we care about here?
1. **Cache Hit/Miss Rate:** 
2. **Cache Size:**
3. **Time To Reconcile After Reconnect:**

## Cache Size

"INSERT FIGURE 5 OF PAPER"

From this figure, somewhere around 50-60 MB of cache would give you 100 percent hit rate!

## Reintegration Time

"INSERT TABLE 1 OF PAPER"

The main takeaway from this table is that **the amount of time sime process runs, isn't in itself indicating how long the reintegration is going to take!**

In brief, the reintegratino time depends on:
1. The amount of IO, not execution time
	   This is because execution time involves computation on client, and that varies heavily epending on what the client does,
2. Amount of bytes that have been changed, and not the number of records
	   This is because a record by itself does not show the amount of data for the update. A single record can show a slight change, or to write a huge binary!

## Conflict Time

"INSERT TABLE 2 OF PAPER"

One main tkeaway frrom the table is that (at least in during the time the paper was written), two different users updating the same file back-to-back is **very rare!**
- Usually, the same user is back-to-back writing during 99.5 percent of cases
- Even when two differnt users are doing writes, there is periods of days between them!

## How Are Things Now?

- These days, client-side cache is much larger
- Upon conflict, it is much easier to just create a new file instead of patching up conflicts