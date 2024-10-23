# The Google File System

Last lecture, we talked about how RPC abstraction works:
- It simplifies writing applications
- Spares the programmer from directly interacting with sockets
- Handle failures, without needing the client to handle all cases by themselves (but they might forward some errors up to the user at times)
- Provide consistency models (at-least-once, most-once, and exactly-once by combining the two and filtering repeated requests)
- Applications don't need to be written in the same language (even the same version!)

Today, we will discuss how RPCs provide an abstraction of a distributed system that can help it look like a local file system. We will discuss one of the first, real implementations of such an abstraction, with the Google File System (GFS).

# Intro.

The GFS was designed to provide a familiar interface like POSIX:
- Read/Write
- Create/Delete
- Open/Close

Plus some other specific operations:
- Snapshots: Create a copy of a file really fast
- Record Append: Concurrent append to the same file

The GFS is very large, as such, failures (disk failures, client disconnects, races, etc.) are the norm rather than the exception. As such, GFS was designed to shard files and replicate them across multiple machines.
The main characteristics of files stored with the GFS are:
- Very little locality
- Very large files
- Read-heavy operation

It is fault tolerant, but has a relaxed consistency model in order to provide higher performance. 

![[Pasted image 20240903122657.png]]

# Architecture

GFS is designed with:
- A **single** Master Node: Coordinates operations and keeps file system metadata in memory
- Multiple **Chunk-server** nodes: These talk to the master and receive requests for file access. They run on top of a typical Linux file system and store the actual files. All data transfer would eventually go through them.
- Many clients: They issue requests to the master and receive results from the chunk-servers.

All files are split between 64 MB chunks, stored in chunk-servers. Each chunk is replicated (usually by a factor of 3) and put into separate servers. Each chunk is addressable with a 64 bit chunk *handle*. This handle covers the entire file system, due to no small amount because of the fact that the chunks are very large. However, even a smaller handle would be able to do that, but chunk handles are changed even after updates, therefore we need to leave room for larger chunk handles without necessarily having to exhaust the whole address space.

The clients never download or upload to the master directly, doing that means that performance would be limited by the network bandwidth of the master. Master only keeps the file system metadata, and lets the other clients navigate to the correct chunk-server.

So a typical read would go like this:
1. A client issues a read request, it gives the file name and the particular chunk index that it wants to write to (the client can do that, since chunk size is fixed, so it just divides the byte offset that it wants to read by the chunk size)
2. The master looks up the chunk in the in-memory namespace by walking a B-tree, and eventually gets the chunk handle targeted.
3. The handle is returned to the client with the address of the chunk-server that has the nearest replicas, the master is done with this client request now.
4. The client contacts the chunk-server and reads the file from it.

The chunk size of 64 MB is important, it has some implications:
- Each chunk-server has 2 racks of 80 GB each. Assuming a total of 5000 chunk-servers, we have 800000 GB of data to address. With a 64 MB chunk handle, we would get about 12.5 million chunks in total. Now each chunks would have some metadata attached to it:
	- 64 bit chunk handle
	- 128 bit IPv6 address, with 2 other replicas, so 3 * 128 bits.
Doing this calculation, the entire metadata comes around 700 MB of data, which can comfortably be kept in memory within the master.

The 64 MB chunk is very important here, while it can waste space, it is the main reason that we can even afford to keep this data in the memory of a single master node.

## Master Metadata

There are 3 things that master holds:
1. File and chunk namespace: Names are compressed with prefix coding (this is necessary to fit the data into a single master)
2. Chunk handles: Mapping from names to chunk handles
3. Chunk-server locations

The first 2 elements MUST be persistent, since none of this data exists in the chunk-servers. The system has a 3 factor replication for these metadata. The persistence is done with an operation log that is checkpointed when it exceeds a certain size. Checkpointing is done by
- Using a B-tree for namespace that can be easily serialized for storage.
- When the log becomes larger than, say 64 MB, we:
	- Open a separate thread
	- Load the previous checkpoint B-tree (or an empty one if there is none)
	- Replay the log on the B-tree
	- Serialize the B-tree and store it
	- Flush the first 64 MB of the log

This checkpointing helps with fast startup, especially important when the master is recovering from a crash.

Chunk-server location is not persisted, since the master keeps a keep-alive with each chunk-server that keeps it updated about the chunks that is stored on each chunk-server (this is needed for garbage collection), so eventually, the master will always learn about each chunk-server contents.

# Consistency Model

(LOOK AT THE SLIDE, I COULDN'T WRITE!)

GFS defines two notions:
- **Consistent:** A chunk is *consistent* after a modification, if all clients see the same data regardless of which replica they use.
- **Defined:** A chunk is *defined* after a modification, if all clients see it consistently, and also see all of the modifications done to it in its entirety.

Say that we have two concurrent writes, we need to consistently serialize writes so that the results on each replica would be the same.

![[Pasted image 20240903132724.png|500]]

During a write operation:
- Client issues a write operation to master
- Master replies with chunk-server address and chunk handle (interaction with master is done now)
- Client sends the data that it wants to write to the closest replica (replica A in above)
- Replica A will forward the received data to the other replicas (it is a chain that gets longer with more replicas).
- Once all replicas have received the data, a primary replica (that is leased for each chunk by the master and changed very slowly when no writes are happening) will inform the client that the write is done.
- If a replica dies, we repeat steps 3-7.

# Snapshots

(LOOK AT THE SLIDE, I COULD NOT WRITE!)

# Replica Placement

When creating replicas, we do it to:
- Create new chunk for new files or expanding ones
- Re-replicate (when replication factor goes up by the user)
- Rebalance disk utilization

The main constraints for *where* a chunk should go is done by considering:
- A chunk must go to at least 2 different racks
- Try to even-out disk utilization, which is tracked with the keep-alive messages received from servers

Each chunk is also given a version number that is persisted within the master disk and the chunk-servers. 
- If the version in master is HIGHER than that of a chunk server, the chunk is stale, it needs to be garbage collected
- If the version in master is LOWER, master reads the chunk and updates itself

## Checksums

Chunks are broken into 64 KB blocks with 32 bit checksums. During reads, the master will verify the checksums and if the chunk-server notices that a checksum is wrong, it reports to the master. We add checksums on top of replications to:
- Prevent corrupted data to be replicated
- Give protection against disk interface corruption to re-replicate a chunk if a replica is corrupted 

# Bottlenecks

- The master is very unlikely to be a bottleneck, unless clients constantly create/delete files
- Hot-spots are possible, especially when multiple clients try to read the same chunk (say, synchronously reading an executable file). These can be fixed by storing such data with higher replication count.