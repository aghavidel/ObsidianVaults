**Paper:** Chain Replication for Supporting High Throughput and Availability

# Chain Replication

This paper proposes the design of a KV store that supports update operations (i.e. NoSQL systems). A client can issue query operations that are more general.
One thing that makes them more general is possible non-determinism, and as such, **they cannot be implemented with replicated state machines**.

The main motivation is high availability.

## Background

Take a look back at RAMCloud, we need at least 3 backups for writes, which means that we can guarantee that data survives with 3 failures and allows reads to be safe.
Consensus is only available with $\frac{N-1}{2}$ failures on a system of $N$ systems.

In Chain Replication, the main selling point is that the system stays available even with $N-1$ failures. As long as a single replica lives, we will serve requests. The main tradeoff here is that the system can continue running with very little replication, and as such, data can be lost if conditions are really bad.
On the other hand, we guarantee that for the data that we *don't* lose, we are consistent.

### Normal Chain Replication

![[Pasted image 20240926122023.png|500]]

A chain consists of a `HEAD` node and a `TAIL` node. 
- The head is the only node that receives write operations and updates.
- The tail node is the only one that is allowed to reply to queries (even updates issued to the head)
- Requests from the head are forwarded to the tail over a reliable FIFO link.

This has some nice properties:
- When the tail replies to an update, we can be sure that the tail and all nodes behind it know about the updates.
- The tail linearizes reads, so we still have linearizability.

### Failures

We should look into 3 scenarios:
- Head fails:
	- In this case, the master will notice failure and elect the next node in chain to be the head.
	  The previous head can become a zombie because of this, as such the new head must be configured to reject all updates from anyone other than itself.
- Tail fails:
	- The master again detects failure and elects the ancestor of the previous tail to be the new tail. The previous tail can once again become a zombie. We need some leasing mechanism to prevent that from keeping in there.
- A node within the chain fails:
	- We must update its predecessor to tell the ancestor of the failed node that it is the new predecessor.
	- We also need to keep track of inflight requests in the head. If the a node in the middle of the chain dies, then the nodes further down the chain may not receive any in-flight update. So we need to keep track of them and resend them if needed. **Only the tail can ACK updates to the head and remove them from the set of in-flight updates**.

## Differences With Primary-Backup Scheme

- This is in general slower. since the length of the chain directly determines latency.
- For the particular case of the head failing, then query processing can still continue without any interrupts. Update processing will be unavailable for only 2 message deliveries (these 2 are a message broadcast from the master to the new head (and its successor?, **look into why, professor thinks that is unnecessary**) and the update to all clients of the previous head to minimize zombie requests).

![[Pasted image 20240926125427.png]]

0                                                                                              