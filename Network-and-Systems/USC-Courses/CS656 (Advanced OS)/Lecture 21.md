**Paper:** [Wide Area Cooperative Storage with CFS](https://dl.acm.org/doi/pdf/10.1145/502059.502054)
# Distributed Storage

In this paper, we see the design and implementation of a file system that lives on a cluster.
## Background: Distributed Systems

Why even go this route? Why machines connected over a network?
1. At some point you can't just use a single machine for storage
2. Going beyond the available processing and IO for aggregate load just is not possible on a single machine
3. If latency is a problem, then using machines that are closer to the clients might be important

We have seen this used, with different degrees, in our previous papers. Like in Difference Engine, Sprite and Elephant FS. Our big blind-spots that we had though are:
1. **Permanent Failures**, what if a machine dies and never recovers? For that, we need failover policies, and that necessary requires to use distributed systems at scale.
2. **High Computation Need**, what if a single CPU just does not suffice, similarly what if a single storage or RAM cannot hold our data?

A more important aspect of distributed systems though is that it allows us to create reliable software, using unreliable hardware. There is also a motivation in terms of cost, as price of equipment does not scale linearly with hardware capacity (doubling the cores on a CPU, probably costs **more** than double the money).

Another big motivation also is Geo-Distributed servers, again for availability and latency constraints.

## CFS

The main idea is, spread the data across different user machines. Contrast this with storing data on data centers and accessing them over the network.

So what is the point? 
- The main benefit is that this is much cheaper, all we need are user devices, not a whole data center or an expensive LAN.
- It is also probably more resilient to data loss. A whole data center can be severely disrupted due to power loss or an earthquake, but it is not very probable that whole fleet of users get lost.

There are drawbacks though:
- Beside lower probability of data loss, all other failures are more likely! 
- A lot of trust is needed between these stranger devices.

### Challenges

There are 3 main fronts to deal with:
- How to balance load across storage nodes
- How to actually *find* these copies
- How to optimize download performance

Let us start with the second one, actually finding the copies.

A strawman approach (used by Napster and early version of BitTorrent), maintain a dictionary that maps file names to IP addresses in a centralized machine. 
This has the big problem of having a giant red mark on its head, if it goes down we are screwed. There is also the fact that at large enough scale, this will become a performance bottleneck. 

So this won't work.
Another approach is like this. Make that dictionary *distributed*, given a file name, calculate some hash of that file name and then find the remainder modulo the number of servers in the system. The number is the ID of the node that holds the file, and we need only know how to reach that node.

To reach the node, we need to give IDs to each node that participates in the network, and then, somehow, make some addressing between nodes (for example, we can maintain a distributed tree among nodes that tells them where each node is somehow).

The big problem with this system though, is that it is only nice when things don't change, if a node joins or leaves, we need to update all of this data and as such there will be a large spike in data churning load.

This is a problem not really with the scheme, but with the *hash* itself. If the modulo base is changed, a lot of data will have to move, since many of the previous hash value need to be updated.

### Background: Consistent Hashing

As always, engineers are pretty bad with names, as such here, the "consistency" really does not mean anything (or at least not as we know it now, originally, it just mean that it had a 'consistent' value).

The idea is like this, keep the hash values on a circle (i.e. the whole hash space is a ring), and then, determine the spot of each server on this ring based on its ID.

![[Pasted image 20240415093944.png|250]]

We iterate this ring clockwise, and we put the constraint that a server $S_i$, will have to handle all hashes that fall in the interval $[S_{i-1}, S_i)$, so from its predecessor (inclusive) to itself (exclusive).

We define the following:
- The **predecessor** of a node on $S_i$ is the first node $S_{i-1}$ that we can see anti-clockwise.
- The **successor** of a hash value $h$ is the first node $S_i$ that we encounter once we move from $h$, clockwise.

A file that hashes to the value $h$ will be stored in its successor, and each node will maintain data about who is its predecessor, and who is the successor of the data in front of it.

So why bother? This is really good for handling join/leave events:
- When a node joins, then only its direct successor needs to modify its files, in particular, it must relinquish the data that is behind the newly added node. Its previous predecessor also needs to update its address data about who is its successor to point to the new node.
- Similarly, when a node leaves, then its files will have to go to its successor and its predecessor needs to update its address data.

The main benefit of this is that only the nodes that directly come before or after the leaving/joining node must even know that something happened, and this makes it very scalable.

So how to actually use this? Well, the problem is that we still need to have addressing data, and the most basic form would be to keep who is the successor, and then if given a file, we didn't have it, then forward it to the successor.

![[Pasted image 20240415095322.png|400]]

This is really bad though, it has $O(N)$ worst performance. We need to improve this.

The main thing that we are wrestling with here is creating a Distributed Hash Table (DHT). This paper designs a pretty efficient DHT called Chord, and the main idea is to convert these linear searches to binary searches, so we get $O(\log N)$ performance. We won't discuss how the table is designed (not that you need to know, Dr. Saleh already saw to that :) ), but the main idea is that each node maintains a **Finger Table** that is of $O(\log N)$ size, which notes the successor of the key in exponential steps (so if the node has ID $S_i$, then it keeps track on who is the successor of $S_i$, $S_i + 1$, $S_i + 2$, ... $S_i + 2^{\log N - 1}$), which allows us to make big jumps across the hash space to see where a file is located.

So this handles the routing part, but there is still a *Load Balancing* problem. If the files are distributed randomly, then popular files will burden their nodes, since they have to handle a very large number of requests.

Also, we are assuming a client can keep a single file, and that just may not be realistic.

So the main idea is that we chunk each file into blocks, and then we choose $k$ replicas for each block and this gives better performance and load balancing (see Balls-and-Bins).

There is still another problem though, not all machines have the same power, so we need some way to scale for each machine. Chord implements this by implementing *virtual nodes*. A machine can have multiple virtual nodes, each with their own ID, and thus, they can participate more in the network by using more virtual nodes.

## Evaluation

TBD