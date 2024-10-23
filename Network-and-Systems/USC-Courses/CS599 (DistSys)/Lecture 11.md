# CURP

The main goal for the paper for this lecture is to reduce the latency submitting requests from a client to a replicated server.

- With primary-backup replication, we almost always have **2 RTTs**:
	- One from client to primary + ACK of operation getting done
	- One for broadcast from primary to all backups
- With Paxos, there is **3 RTTs**:
	- One from client to a node + ACK of operation getting done
	- One RTT for propose
	- One RTT for accept
- Paxos can be optimized slightly, in particular, for *Fast Paxos*, we can let:
	- Clients broadcast to all other replicas, and wait for ACK from majority
	- Each replica screams at the others on an unreliable network about whether or not it got a value from client (this needs only half an RTT)
	- This gives **1.5 RTT**
	- This does NOT work as-is when there are concurrent requests from clients, for that we need *generalized Paxos*, which is pretty cryptic of an algorithm.

The goal is, can we drop this lower? Ideally to just **one RTT from client to system + ACK of operation being done?**

## CURP

The main observation in designing CURP, is that many operations are commutative, as long as they happen within a short enough time interval.

>[!REMINDER] Commutative Operations
>It is exactly as you think. It means that the order of executing them does not change the outcome of them as a whole.
>A trivial case is operations on completely separate data structures.
>In general though, deciding whether or not some operations commute without knowing what data they update is not possible (SQL `UPDATE` commands are  a prime example of this).

The idea is:
- Assume we have a bunch of commutative operations. As long as we can finish them soon enough, we can get another set of commutative operations for the next round. 
- Clients maintain a list of *witness* machines. You need as many witnesses as the amount of server failures you want to tolerate.
	- A client broadcasts to both the primary and the set of all witnesses.
	- A witness can reject clients operations if they are not commutative with the operations already on the witness.
	  If this happens, client MUST ask the primary to synchronize its state directly. Doing this adds another RTT and we go back to the 2-RTT model, but hopefully it happens rarely.
	- If majority of witnesses accept a client request, we can assume the operation is done and go on with our life. We should read only from witnesses as long as synchronization is not done.

### Crash Recovery

When a master crashes, we should:
- Let the primary recover by reading from backups (do exactly what they would do if CURP did not even exist)
- Choose a **single** witness, lock it so that it no longer accepts operations, and replay operations from it and wait for synchronization.
- (CHECK THIS!) Flush all witnesses, and return to normal operation.

### Avoiding Duplicate Execution

During replay from witnesses after a recovery, we could execute operations twice!
We need some linearizer to filter out these operations. RIFL would do just fine in preventing duplicate executions.

## Discussions

As we mentioned, operations tend to be commutative only for small time windows. As such, CURP would not help at all if the system is just too slow for backing up operations.

If you also set the condition that you would only read from the primary (which you primary should
)

There are disadvantages:
- You need beefier clients to handle broadcasts to witnesses along with the primary
- If your work-load is skewed so that the number of commutative operations is too small, the performance drops.
- Geo-replication makes things difficult for this, as broadcasting to replicas becomes really difficult from client side.

In general, each backup must have a witness associated with it. 

### Garbage Collection

READ FROM LECTURE, I COULD NOT WRITE!

## CURP For Consensus Protocols

Until now, we have only studied P-B systems, we need to also consider consensus protocols.

The semantics of CURP are the same, but there is a change. Client CANNOT return with just witnesses. We need a **super-quorum** to return.

>[!REMINDER] Quorum vs Super-Quorum
>When we want to tolerate $f$ failures, we define the following:
>- **Quorum:** Means to have ACK from $f+1$ nodes. 
>- **Super-Quorum:** Means to have ACK from $f+\frac{f}{2}+1$.

