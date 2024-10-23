# Two-Phase Commit and Its Optimizations

The main topic of concern for this lecture is *multi-object transactions*. If you have taken a database course, you would know of the *ACID* model of transactions. We want a transaction to be Atomic, Consistent, Isolated and Durable.

In more familiar terms, we want them to be:
- **Atomic:** A transaction does all of its operations, or nothing. If a transaction has a chain of operations $O_i$ for $i \in \{1, 2, ..., n\}$, then the transaction eventually MUST execute all of the operations $O_i$, or rollback all of them in case any $O_j$ fails.
  Note that there is no constraint that prevents the results of any of the intermediate $O_i$ from being revealed to the outside world.
- **Serializability:** Two transactions $T_1$ and $T_2$, are always executed one-by-one. We will NEVER see intermediate results.
  A system that is both Atomic and Serializable, would be very predictable. Given a series of transactions $T_i$ and the order of their execution:
	- If all $T_i$ succeed, the result is exactly the same as if we executed them one-by-one on  a single instance.
	- If any $T_i$ fails, as long as the results of the other transactions does not depend on it, we can just repeat the transaction until it succeeds.

## Distributed Transactions

In its simplest form, transactions are done by getting a transaction ID from a master server (the `Tx ID`). Then the `Tx ID` is used to lock objects when reading/writing them.
- If commit finishes, then all if well.
- If commit aborts, all of the results are rolled back by getting rid of any update labeled with the above `Tx ID`.
- Almost always, a client has a cache, which keeps objects that were read for the transaction. If the server locks the objects, then it is easy to just execute all writes on the cache, and then push the results to the actual server.
  Another way is *Optimistic Locking*, where the data is read once into the cache with a version number. Before pushing the data, the object is locked for a moment (before this, it was free), and the version is checked. If the version is the same as before, commit is done and the lock is freed. If the version is changed, an error is returned to the client to retry and the lock is freed.

>[!FAQ] Why Use Optimistic Locking?
>It has the benefit of letting read operations continue when a transaction is going on. This will not violate serializability, but means that each write has a higher chance of failure due to concurrent access.

## Plain Two-Phase Commit (Presume Nothing, `PrN`)

A topology for this protocol would include a series of *Cohort* servers, which could in theory be a whole  primary-backup system that looks like a single node. One cohort assumes the role of a *coordinator*, the first thing that kicks off the transaction (likely facing the client requests).

A coordinator always marks the first operation of a transaction. If a cohort dies, comes back after a transaction is started, and sees the commit mid-way, then it would immediately abort it when it notices that it has never seen the transaction start.

- A coordinator sends `REPARE` to all cohorts.
- A cohort responds with a `COMMIT-VOTE` or `ABORT-VOTE`. The results is made persistent on the cohort.
- The coordinator sends the outcome, either `COMMIT` if everyone is ready, or `ABORT` if anyone rejected. The result is again made durable.
- The cohorts will then ACK the outcome.

Persisting into disk is referred to as a *Forced-Write*. The distinction is made because usually we would use a disk for this, and disk don't actually ACK writes immediately, they would buffer it and then ACK it later. A forced write means that the ACK **MUST** be collected before doing the operation of the next steps.

Thus, in `PrN`, we MUST make the outcome durable on the coordinator before sending any commit/abort message to cohorts.

### Optimization: Presumed Abort (`PrA`)

Due to how the protocol is designed, if a transaction is not completed when we recover from a crash, we can be sure that it was aborted. As such, there is no reason to make aborted transactions persistent in the log, only committed transactions need to be persisted in the log.

### Optimization: Presumed Commit (`PrC`)

Similarly, we can change the protocol in such a way that notes that logs are only finished for transactions that actually commit. We force a write for the initiation of each commit, and another forced write for actually committing a transaction.
When a transaction aborts, we end the log entry for the transactions.

The good thing is that in this model we don't need to get ACKs for commits in the second phase. As long as the outcome is logged, we are safe. The price is that a forced initiation record is required for ALL transactions (even those that will abort).

## A New Presumed Commit (`NPrC`)

We can also further optimize this type of transaction. Assume monotonically increasing transaction identifiers and maintain the set of in-flight transactions. To do that, maintain the following sets:
- `COMM`: Set of `Tx ID`s for transactions that we know are committed
- `TID_l`: The largest `TID` for a transaction that may have been executed
- `TID_h`: The largest `TID` for a transaction that was initiated

The set of IDs between `TID_l` and `TID_h` that are not in `COMM`, are in-flight commits. If we crash and recover, the set of IDs:
$$
\text{IN} := [\text{TID}_l \; .. \; \text{TID}_h]\; - \;\text{COMM}
$$
Can be safely presumed to have been aborted, and we can then bulk-initiate a transaction for all of them at once after recovery.

>[!FAQ] Why Is $\text{IN}$ Presumed Abort?
>If the persistent log of an outcome exists, then the ID must be in $\text{COMM}$.
>On the other hand, no initiated transaction can exist outside of the range from $\text{TID}_l$ and $\text{TID}_h$ that is not finished. If we have the initiation log, but not the commit log, then the commit result was not persisted anywhere.

In the above:
$$
[\text{TID}_l\;..\;\text{TID}_h]\; := \{t\;|\;\text{TID}_l < t < \text{TID}_h\}
$$

Note that all cohorts need to log the commit preparation durably, since if they don't they wouldn't know if they did or did not apply the commit when they crash. Remember that the coordinator only lets you know if the commit wasn't aborted, it will not tell you if you actually applied it!

"I HAVE SOME DOUBTS ABOUT MY UNDERSTANDING OF THE SLIDES, READ THE PAPER!"


The idea is to get rid of the first forced log write for initiation, to reduce the latency of starting a transaction. 

![[Pasted image 20241015132951.png|500]]

The protocol itself messages in exactly the same way as `PrC` would, but the difference is with what we write in the logs.

