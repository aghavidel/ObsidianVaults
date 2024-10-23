**Paper:** [A Low-Bandwidth Network File System](https://dl.acm.org/doi/pdf/10.1145/502034.502052)
# A Low-Bandwidth Network File System

This paper was one of the first formal implementations of file fingerprinting and chunking for low bandwidth usage.

## Intro.

We have seen file caching as a technique for many different reasons. This paper provides a new argument for file caching, which while more subtle, is now completely ubiquitous.

The argument is that for ***interactive*** applications, it is much preferable that IO remains local (like individual key strokes) and only file results are remote.

For example, imagine a remote text editor. An editor is just a user typing things into a system and seeing them render on a local interface (say, a monitor). So we want these to transmit fast, so we keep them local (i.e. they effect the cached instance, not the remote one directly), and we just send it to the server occasionally.

This is quite different compared to Coda or Sprite:
- In those, the file server was hosted outside the local network
- The bandwidth between the two endpoints is much less available in this new case, since it traverses a WAN instead of a clean and operational LAN

The solution here, the Low Bandwidth File System (LBFS) operates in this region.

## LBFS

In brief, the operation work-flow is:
- Client downloads a copy of the file when doing `fopen`
- All updates are applied locally and backed by a persistent storage.
- Client uploads the updated copy from cache to server on `fclose`

This system provides *Close-to-Open Consistency*, a much simpler model, which guarantees that **when a user opens a file, it will then see the updates of all clients who have closed the file before it**.

So before we go on, lets consider some things more carefully:
### Uploading A File

Following our previous works, there are 2 options apparently:
- Compress file before uploading
	- Does not work well, since even if you change a single byte, the whole things needs to be compressed and uploaded
- Upload only a log of writes to a file
	- Copy at the server might need to be updated soon after opening
	- Doing concurrent writes with multiple clients seems troubling
	- This only works for uploads, we need to think of something for downloads as well

The main idea in LBFS is to leverage the similarity between files:
- For uploading on close:
	- Use similarities between copies at client and server
- What if client has updated file significantly?
	- Updated file may be similar to other files at server
- For downloading on opening:
	- Client may have an older copy of the file cached
	- Client may have other similar files (for example, executable files can be similar because they are making use of the same libraries)

The question however is, how exactly do we *compare* entire files?
One way would be to:
- Pick a fixed Chunk Size, say a few KBs
- Compute the SHA-1 hash of each chunk
- Compare chunk hashes. If they match, then don't update!

Unfortunately, this really does not work well. The reason is that *adding* data will shift all subsequent chunks in the same file, all of those chunks will be changed as well! Fixed-sized chunking just won't work, we need to have variable sized ones that are content-aware.

One solution for this is Rabin Fingerprints. which we should probably discuss in more detail, but the high level abstraction of it is:
- Scan the file from start to end for each chunk
- At each byte, compute the fingerprint (some hash) over the last 48 bytes
- If the 13 LSB bytes of the fingerprint match a sentinel value (a fixed constant defined by the hash function), then this is the end of the chunk, move on to the next

Graphically:

![[Pasted image 20240226144354.png|500]]

Here:
- We start with at the top with (a)
- We make an edit to the chunk $C_4$ and get a bigger chunk $C_8$
- We make another edit on $C_5$ and get $C_9$ and $C_{10}$
- We make a final update to both $C_2$ and $C_3$. Depending on what is happening, this can cause the two chunks to merge or change independently

### Downloading A File

Again, graphically:

![[Pasted image 20240226144641.png|500]]

This is quite simple, just download the hash of each file first and then get the actual data.

### Conditional Writes

For uploading, we need to be careful. While downloads were simple, uploading is a bit more complicated:

![[Pasted image 20240226145801.png|500]]

The main concern here is the following scenario:
- Client A opens file and starts editing
- Client B opens file and starts editing
- Client B closes and updates the file
- Client A closes and updates the file, but a conflict happens with the updates already done by B

We again face a two-phase commit, and so we turn to our old friend, **memory shadowing**. So to implement this, we create a temporary shadow copy on the server side and we populate it with the data from the client. The only difference here is that we actually first send the hash to make sure the data even needs writing in the first place, and *then* we commit it.

This is done with the `CONDWRITE` and `OK/HASHNOTFOUND` messages. For every missing chunk, the client would send a `TMPWRITE` message to populate the temporary copy with new data. Once the client has no further data, it will commit by sending a `COMMITTMP` message and waiting for an `OK` from the server.
Upon receiving the `OK`, the file is closed and we are done.

## Evaluation

Of course we really should consider the bandwidth usage (its literarily in the name, so it should go without saying).

### Using Similarities

The authors used the source codes of Emacs as well as some random MS Word documents to illustrate how much redundancy exists in these files:

![[Pasted image 20240226152241.png|600]]

It is quite self-explanatory, the only interesting tidbit here is the second row, there exists 38 percent redundancy in the source code *Just By Itself!* That's quite insane.

### Bandwidth Utilization

![[Pasted image 20240226152819.png|500]]

Before moving on:
- NFS does no caching at all, so writes are greedy
- The Andrew FS (AFS) defers writes
- Leases + Gzip implements file compression on top of file systems
- LBFS + new DB means that we don't have previous hashes to make use of for similarities 

For downloading:
- Really only big difference is NFS, and that happens because there is no caching on the client so we use a huge amount of bandwidth every time!

For uploading:
- NFS performs the worst, since writes cannot be deferred
- AFS performs next best, but it still makes no use of redundancy nor similarities between files
- Using Gzip shows dramatic improvement, showing how much these workloads are actually compressible! 
- LBFS with new DB still performs better than Gzip during upload, since it can do less writes, since it recognizes some writes are already redundant
- Full LBFS uses redundancy among all of its files, so its even better

### Performance

![[Pasted image 20240226153421.png|500]]

Not much to discuss here, but it is astonishing that LBFS approaches the performance of a LAN on a WAN!

There is also the question of what actually bottlenecks the performance:

![[Pasted image 20240226153626.png|700]]

Since the network utilization during upload is lower, LBFS copes much better when there is a hard cap on total bandwidth, keeping the execution time low.

## Broader Adoption

This work implements techniques that now run literarily *everywhere*. Content based chunking is a technique that can be used many different scenarios, but it is critical for Automatic Backup Systems. Why? Because backups naturally have quite a bit of similarities!

There is however, some growing security concerns.

- These systems look at many different files at the same time for efficient chunking, that seems ripe for doing bad things
- Beyond that, these systems are known to be vulnerable to Side-Channel attacks, since for example, if a file does not exist in a server it takes much longer to upload compared to if it does

The bigger problem is that much of the data these days can be encrypted. So even through the plaintext are similar, the ciphertexts can be different! This is one of the main reasons that storage hosts dislike encrypted files!

Maybe someone should look into what we can do about that ...