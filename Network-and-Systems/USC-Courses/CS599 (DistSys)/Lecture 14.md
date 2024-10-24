
Today we will see how Google implemented a mostly lock-free distributed database. 
As we have seen until now, a distributed storage system needs two parts:
- **Concurrency Control:** Order transactions across machines with different shards/replications
- **Replicated State Machine:** For fault tolerance 

As we have seen until now, there is a tradeoff between consistency and performance. What most systems want is a consistency guarantee that is intuitive, and the one that is most intuitive as we have seen, is *Strict Serializability*.

Strict Serializability means to combine linearizability and serializability:
- If a transaction is committed before another transaction starts, the result of the first will be seen before the second
- No intermediate results from a transaction are externalized

This guarantee is also called *External Synchronization*, which we actually saw in CS 656.

**Problem:** Google's databases are geo-replicated, and as such, there is high delay for write operations. If we lock objects for writes, we will limit the throughput of the system heavily since we would have to deny other reads.

We want to:
- Make read-write transactions reasonably fast
- Make read-only transactions as fast as possible (high priority)
- Make the whole thing externally synchronous

>[!FAQ] Didn't We Just Learn 2PC?
>2PC still relies on locks to make sure that ongoing transactions don't race on the same replica. As such, if two *read only* transactions land on the same data instance, one of them will have to wait. Waiting here means that the node will take time to answer the `PREPARE` message and vote for commit or abort.
>
>This is even worse here, since Google's Spanner is supposed to be Geo-Replicated! Even reaching a coordinator for 2PC can be time consuming, since the coordinator can be hundreds of milliseconds away from where cohorts are!!

# Spanner: Google's Geo-Replicated Database

Before discussing Spanner, let's discuss the case for read-only transactions.

## Read-Only Transactions

Take RAMCloud for instance:

![[Pasted image 20241024013859.png|400]]

- A transaction from the client is emitted to the server, it will want to log multiple `PREPARE` entries on the server.
- All `PREPARE` logs are made durable in the backups. During this time, all objects that we read are locked.
- After the above, all `DECISION` logs also need to be made durable when they come.

Thus, in effect, we have **1 RTT + 1 FORCED LOG WRITE** of time for which an object is locked, **with the RTT of a WAN!**. If any read-write transaction comes in during this time, the performance would be awful.

How to handle this?
There are multiple ways, two important ones:
- **Optimistic Locking:** Version each object, when the transaction starts, read all objects, but before returning, re-check the version (no locking is involved).
  If version has changed, abort, if not, return.
  Returns fresh data always, but may abort. (This is what RAMCloud did)
- **Snapshot Isolation:** Keep consistent snapshots of data for certain timestamps. For each read transaction, demand a minimum timestamp and return the oldest snapshot.
  Returns minimally stale data that is guaranteed consistency and never aborts.

We will focus on Snapshot Isolation, since that is what Spanner uses. For example:
- `TX-1`: Let `A=1` and `B=2` for timestamp `t=1`
- `TX-2`: Let `A=3` for timestamp `t=3`

If a read-only transaction with timestamp `t=2` comes, then we will always return `A=1` and `B=2` without any locking by reading any replica. If the user explicitly says what timestamp to use, then great, if not, *somehow* we should pick one for the user.

Spanner deals with the last part (how to pick timestamps) as we will discuss.

>[!IMPORTANT]
>Most of the optimizations that we will discuss here only really make sense under the assumption that the workload is heavily read dominated.
>This makes sense in many real world cases, but there are times where it just does not work that well. Spanner isn't one of those cases though!

## How To Version Data With Timestamps

The goal here is Strictly Serializable read-only transactions without locking, 

We assert for all operations that:
- When committing a write, tag it with the current physical time
- When reading the system, check which writes were committed before the given time and then choose the appropriate snapshot

To prevent this from falling apart, we need some time invariants.

>[!IMPORTANT] Requirement for External Synchrony
>If a transaction $T_1$ commits before a transaction $T_2$ starts, the timestamp for them ($s_1$ and $s_2$ respectively) must obey $s_1 < s_2$.

Assume we have a perfect clock. Reading the clock gives us timestamps, and that naturally gives us a consistent ordering of values (a total order in fact!), thus we just read the clock when we commit and output it as the timestamp.

On a distributed system though, things become complicated. In particular, distributed systems don't have fully synchronized clocks. Differences can be up to a few milliseconds, and on a system as large as Spanner, you can easily get race conditions because of timestamps that have incorrect ordering, we need to somehow cope with this lack of full data.

To see how this can break the system, look at the following:

![[Pasted image 20241024044611.png|600]]

In the above, $T_{\text{abs}}$ is a perfect wall clock. Calling `now()` will sample it instantly, but there is a delay here, since the call is executed remotely. So when we call `now()` and we get 6, the true time is actually larger than that!

In the above, the call for $T_2$ returns something that is within $[5, 10)$, and the call for $T_1$ returns something within $[8, 15)$. As you can see, the intervals **overlap**, and that is the thing that will cause a problem!
The reason for this is that it is possible to have an execution, where even though the user issues the transactions in the correct order, the time read for $T_1$ returns too late and the time read for $T_2$ returns too early, and so we end up with timestamps that imply the operation order was reversed!!
## Spanner Design

In Spanner:
- We have multiple shards that are geo-replicated
- Each shard, has multiple replicas which are maintained with Multi-Paxos

Spanner has a *global wall-clock* referred to as **True Time** or $TT$.
The data structure $TT$ is a volatile variable that has two fields, `earliest` and `latest`.

**Note:** Here, we use $e_{event}$ as the absolute time of the event $event$. Here, absolute time means in relation to a perfect clock running alongside the system.
We have 3 events that we care about:
- Event $\text{start}_i$, where a transaction $T_i$ actually gets issued by a client and Spanner realizes that it exists.
- Event $\text{server}_i$, where the result of a transaction $T_i$ gets applied (i.e. you probe a memory location somewhere in the system and see the result).
- Event $\text{commit}_i$, where a transaction $T_i$ returns its result to its client.

Causality mandates that $e_{\text{start}_i} < e_{\text{server}_i} < e_{\text{commit}_i}$.

>[!IMPORTANT] Invariant $1$
>For an observation of $TT$ issued at $e_{invoke}$, we guarantee that the result that we return (call it $t_e$) satisfies $t_e\: . \text{earliest} \leq e_{invoke} \leq t_e\: . \text{latest}$.

Google implemented the backend for $TT$ over a series of GPS machines, quartz clocks and atomic clocks on satellites. The error of $TT$, defined as $\epsilon := \frac{TT\:.\text{latest} - TT\:.\text{earliest}}{2}$ tends to be upper bounded by around 10 milliseconds, and has a  mean of around 4 milliseconds.
Google reports that it typically exhibits a saw-tooth like behavior.

With the above case, we define:
$$
\begin{align}
TT.\text{after}(s) &:= s < TT.\text{earliest} && (\text{Timestamp $s$ has passed}) \\
TT.\text{before}(s) &:= s > TT.\text{latest} && (\text{Timestamp $s$ is in the future})
\end{align}
$$
Assume $T_1$ and $T_2$ are defined as in the requirement. We assume that the transactions satisfy $e_{\text{commit}_1} < e_{\text{start}_2}$, which means that we assume transaction 1 is done before we move on to transaction 2 (this means that the client actively blocks until it gets the result of transaction 1 before it even issues transaction 2).

>[!IMPORTANT] Invariant $2$
>Never return a result for a transaction to the client, unless the timestamp for the transaction satisfies $TT.\text{after}$. Thus, for all transactions $T_i$, we **artificially wait** until $TT.\text{after}(T_i.\text{start})$.

In simple terms, invariant 2 enforces that $s_1 < e_{\text{commit}_1}$.
Thus, we have the following chain of inequalities:
$$
\begin{align*}
	s_1 &< e_{\text{commit}_1} && (\text{Invariant 2}) \\
	e_{\text{commit}_1} &< e_{\text{start}_2} && (\text{Assumption}) \\
	e_{\text{start}_2} &\leq e_{\text{server}_2} && (\text{Causality}) \\
	e_{\text{server}_2} &\leq s_2 && (\text{Protocol}) \\
	&\Rightarrow s_1 < s_2
\end{align*}
$$
And we are done!

Again, the key here was the artificial wait that we had to do after getting the timestamp to make sure that it has passed!

![[Pasted image 20241024045440.png|600]]

Note that the value of $\epsilon$ here by itself is not that important, the system would still be correct, regardless of what it actually is. However, the larger the $\epsilon$, the larger the wait times are expected to be, and as such we will benefit from minimizing it!

## Read-Write Transactions

The above fixes everything for read-only transactions without a lock, but what about read-write ones?
Take for example, the transaction that reads a value $A$ and writes $A \gets A+1$, $B \gets A+1$ and $C \gets A+1$. 

- These transactions are known before execution to be read-write, and thus, the read of $A$ will NOT be without a lock. A read lock will be granted to the client for the duration of the transaction.
- The client gets the value of $A$ an buffers it locally. Since it has the lock, it can compute all updates locally (i.e. the new values of $A$ and $B$ and $C$).
- Once the client has all its writes, it will send all at once to the server, and once they are inserted, 2PC will consistently commit the transaction and release the locks.

![[Pasted image 20241024050614.png|600]]

Rather vexingly, this is called *Two Phase Locking (2PL)*, which is weird:
- It is 3 phased actually (i.e. one phase get all read locks, second phase, do a 2PC with the write locks, which is actually 2 phases)
- The 2 phase here just means the *Execute* and *Prepare* phases, where locks are actually granted to the client.

Now again, this is with *perfect clocks*! What about with Spanner?

![[Pasted image 20241024050833.png|600]]

For prepare phase, as we remember from 2PC lecture, participants (i.e. Cohorts) MUST force log an entry for preparing. The timestamp for actually finishing the write commits must be assigned before we can log it of course, these are $ts_B$ and $ts_C$ in the above. Note that they **DO NOT have to actually wait at all if they choose the timestamps well.**

- The participants can choose a timestamp that is lager than any write timestamp that they know, it will work fine.
- The participants MUST report these timestamps to the Coordinator.
- The Coordinator is also a Cohort in the picture above, so it too must commit and get a timestamp. However it needs all the following properties:
	- Larger than any previous write (obviously for Linearizability)
	- Larger than all timestamps reported to it from the participants (again for Linearizability)
	- Satisfying $TT.\text{after}(.)$ 

Many a times, the commit waits can be overlapped with Paxos log writes.
It is also important to note that for read-only transactions, Paxos shards also need to wait. In particular, they must wait until after all previous write timestamps have passed to preserve the consistency of logs.