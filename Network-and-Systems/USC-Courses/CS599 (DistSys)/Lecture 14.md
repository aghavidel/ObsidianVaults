
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

# Spanner: Google's Geo-Replicated Database

Before discussing Spanner, let's discuss the case for read-only transactions.

## Read-Only Transactions

Take RAMCloud for instance:
- A transaction from the client is emitted to the server, it will want to log multiple `PREPARE` entries on the server.
- All `PREPARE` logs are made durable in the backups. During this time, all objects that we read are locked.
- After the above, all `DECISION` logs also need to be made durable when they come.

Thus, in effect, we have **2 RTTs** of time for which an object is locked. If any read-write transaction comes in during this time, the performance would be awful.

How to handle this?
There are multiple ways, two important ones:
- **Optimistic Locking:** Version each object, when the transaction starts, read all objects, but before returning, re-check the version (no locking is involved).
  If version has changed, abort, if not, return.
  Returns fresh data always, but may abort.
- **Snapshot Isolation:** Keep consistent snapshots of data for certain timestamps. For each read transaction, demand a minimum timestamp and return the oldest snapshot.
  Returns minimally stale data that is guaranteed consistency and never aborts.

We will focus on Snapshot Isolation, since that is what Spanner uses. For example:
- `TX-1`: Let `A=1` and `B=2` for timestamp `t=1`
- `TX-2`: Let `A=3` for timestamp `t=3`

If a read-only transaction with timestamp `t=2` comes, then we will always return `A=1` and `B=2` without any locking by reading any replica. If the user explicitly says what timestamp to use, then great, if not, *somehow* we should pick one for the user.

Spanner deals with the last part (how to pick timestamps) as we will discuss.

## How To Version Data With Timestamps

We assert for all operations that:
- When committing a write, tag it with the current physical time
- When reading the system, check which writes were committed before the given time and then choose the appropriate snapshot

To prevent this from falling apart, we need some time invariants.

>[!IMPORTANT] Requirement for External Synchrony
>If a transaction $T_1$ commits before a transaction $T_2$ starts, the timestamp for them ($s_1$ and $s_2$ respectively) must obey $s_1 < s_2$.

For a perfect clock, this is trivial, for a distributed clock system, this is far from trivial.

## Spanner Design

In Spanner:
- We have multiple shards that are geo-replicated
- Each shard, has multiple replicas which are maintained with Multi-Paxos

Spanner has a *global wall-clock* referred to as **True Time** or $TT$.
The data structure $TT$ is a volatile variable that has two fields, `earliest` and `latest`.

>[!IMPORTANT] Invariant $1$
>For an observation of $TT$ issued at $e_{invoke}$, we guarantee that the result that we return (call it $t_e$) satisfies $t_e\: . \text{earliest} \leq e_{invoke} \leq t_e\: . \text{latest}$.

Google implemented the backend for $TT$ over a series of GPS machines, quartz clocks and atomic clocks on satellites. The error of $TT$, defined as $\epsilon := TT\:.\text{latest} - TT\:.\text{earliest}$ tends to be upper bounded by around 10 milliseconds, and has a  mean of around 4 milliseconds.
Google reports that it typically exhibits a saw-tooth like behavior.

With the above case, we define:
$$
	TT.\text{after}(s) := s < TT.\text{earliest}
$$
Assume $T_1$ and $T_2$ are defined as in the requirement. Denote $T_{real}$ as the time returned by a perfect clock for an event. We assume that the transactions satisfy $T_{real}(T_1.\text{commit}) < T_{real}(T_2.\text{start})$.

>[!IMPORTANT] Invariant $2$
>Never return a result for a transaction to the client, unless the timestamp for the transaction satisfies $TT.\text{after}$. Thus, for all transactions $T_i$, we **artificially wait** until $TT.\text{after}(T_i.\text{start})$.

In simple terms, invariant 2 enforces that $s_1 < T_{real}(T_1.\text{commit})$.
Thus, we have the following chain of inequalities:

(THIS IS RUBISH, FIX IT YOU IMBICILE!)

$$
\begin{align*}
	s_1 &< T_{real}(T_1\:.\text{commit}) && (\text{Invariant 2}) \\
	T_{real}(T_1\:.\text{commit}) &< T_{real}(T_2\:.\text{start}) && (\text{Assumption}) \\
	T_{real}(T_2\:.\text{start}) &\leq T_{real}(T_2\:.\text{commit}) && (\text{Causality}) \\
	T_{real}(T_2\:.\text{commit}) &\leq s_2 &&
\end{align*}
$$
