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

The key here is to see, why exactly do we end up with 2 RTTs most of the time?

![[Pasted image 20241024054018.png|500]]

In particular, clients want serializability and durability (i.e. replication). For a PB system, the first RTT is there to partially order things, and the second RTT is for replication.

Now:
- Replication MUST be done, it is very hard to see how we can get away with shortening that without breaking durability.
- Serializability however, can be *deferred*, we need only let the client know of what order things should be before we finish the transaction, not necessarily before replication!

The second point is ripe ground for exploiting, since if we assume operations are *commutative*, any ordering is acceptable!
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
	- The primary says that it is done immediately without actually synchronizing.
	- A witness can reject clients operations if they are not commutative with the operations already on the witness.
	  If this happens, client MUST ask the primary to synchronize its state directly. Doing this adds another RTT and we go back to the 2-RTT model, but hopefully it happens rarely.
	  The worst case here is that:
		- We get the request on Primary, return to client but synchronization gets deferred 
		- We contact the witnesses, get reject by at least 1
		- We have to tell the Primary explicitly to synchronize
	 This essentially adds up to 3 RTTs.
	- If ALL of witnesses accept a client request, we can assume the operation is done and go on with our life. We should read only from witnesses as long as synchronization is not done. We need not explicitly wait for synchronization to finish.

![[Pasted image 20241024054611.png|600]]

### Crash Recovery

When a master crashes, we should:
- Let the primary recover by reading from backups (do exactly what they would do if CURP did not even exist)
- Choose a **single** witness, lock it so that it no longer accepts operations, and replay operations from it and wait for synchronization.
- (CHECK THIS!) Flush all witnesses, and return to normal operation.

### Avoiding Duplicate Execution And Unsynched Data

During replay from witnesses after a recovery, we could execute operations twice!
We need some linearizer to filter out these operations. RIFL would do just fine in preventing duplicate executions.

Also, the primary does not keep track of what is happening in the witnesses, and as such, when a client starts another transaction, it is possible that the primary could return data is not yet synchronized. 
In this case, the primary must block until synchronization is done (effectively it turns to a normal PB system for these requests).
## Discussions

As we mentioned, operations tend to be commutative only for small time windows. As such, CURP would not help at all if the system is just too slow for backing up operations.

There are disadvantages:
- You need beefier clients to handle broadcasts to witnesses along with the primary
- If your work-load is skewed so that the number of commutative operations is too small, the performance drops.
- Geo-replication makes things difficult for this, as broadcasting to replicas becomes really difficult from client side.

### CURP-H

An optimization can be done with CURP. In particular, if we glue together witnesses and backups, we get nice properties (this is what we call CURP-H).

![[Pasted image 20241024060404.png|600]]

One main benefit of this is garbage collection of witnesses.

- In normal CURP, data stays in the witness for at least 1.5 RTTs. 0.5 RTT time for actually replying back to client and potentially 1 RTT at least for synchronization.
  Note that once we return to client, at best, synchronization is started (this would be 1 RTT after the client issued the operation). Thus, another half RTT is used for actually getting the sync request to backup, and another half to deliver the synchronization done message to both the primary, AND witnesses.
  A witness that sees that synchronization is done can garbage collect the associated entries.
- With CURP-H, we get rid of a whole RTT and just end up with 0.5 RTT, which means that we garbage collect right after we issue the synchronization request.
  Why is this safe?
	- Note that here, witness and backup share fate, thus:
		- If backup dies, the witness goes with it
		- If backup does not die, it WILL synchronize.
	 Thus, there is no need to keep data in witness after synchronization request is sent.

![[Pasted image 20241024061056.png|600]]

## CURP For Consensus Protocols

Until now, we have only studied P-B systems, we need to also consider consensus protocols.

The semantics of CURP are the same, but there is a change. Client CANNOT return with just witnesses. We need a **super-quorum** to return.

>[!REMINDER] Quorum vs Super-Quorum
>When we want to tolerate $f$ failures, we define the following:
>- **Quorum:** Means to have ACK from $f+1$ nodes. 
>- **Super-Quorum:** Means to have ACK from $f+\frac{f}{2}+1$.

In particular, like CURP-H, each replica has a witness attached to it. Assume we have $2f$ total replicas with a single leader (the witness in leader is inactive, it does nothing).
For normal execution:
- Client shouts requests at every replica
- Leader returns execution in 1 RTT and gives acceptance
- Acceptance from $f + \frac{f}{2}$ witnesses is necessary

So why super-quorum is needed?
Note that  witnesses keep entries, even for ones that might have failed! Thus, if two transactions start at the same time, one of them being destined to fail, we will end up with conflicting entries in witnesses.
Assume $f$ servers fail, we end up with $f$ remaining servers, and we must make sure that only a strict minority of servers here have witnessed the failed transaction, thus we need at least $\frac{f}{2}+1$ witnesses at our side to prevent that.
Thus, we get $f + \frac{f}{2} + 1$ as needed.