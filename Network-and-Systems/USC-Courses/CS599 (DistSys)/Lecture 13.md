# Two-Phase Commit and Its Optimizations

The main topic of concern for this lecture is *multi-object transactions*. If you have taken a database course, you would know of the *ACID* model of transactions. We want a transaction to be Atomic, Consistent, Isolated and Durable.

In more familiar terms, we want them to be:
- **Atomic:** A transaction does all of its operations, or nothing. If a transaction has a chain of operations $O_i$ for $i \in \{1, 2, ..., n\}$, then the transaction eventually MUST execute all of the operations $O_i$, or rollback all of them in case any $O_j$ fails.
  Note that there is no constraint that prevents the results of any of the intermediate $O_i$ from being revealed to the outside world.
- **Serializability:** Two transactions $T_1$ and $T_2$, always *appear* to execute one-by-one. We will **NEVER** see intermediate results.
  A system that is both Atomic and Serializable, would be very predictable. Given a series of transactions $T_i$ **and** the order of their execution:
	- If all $T_i$ succeed, the result is exactly the same as if we executed them one-by-one on  a single instance.
	- If any $T_i$ fails, as long as the results of the other transactions does not depend on it, we can just repeat the transaction until it succeeds.

We also introduce a stronger conditions, **Strict Serializability**. As you might have noted above, the order of execution for each transaction does not seem to be forced, and you are correct! Indeed, that is *NOT* part of serializability by default. Strict Serializability aims to rectify this. In particular, the order of the actual execution of each transaction MUST match the order which the user actually provides them.

>[!EXAMPLE]- Atomicity vs Serializability vs Strict Serializability
>Consider the example where we have two transactions, $T_1$ and $T_2$:
>- $T_1$: Deposit 200$ to account $A$
>- $T_2$: Transfer 100$ from account $A$ to account $B$
>So $T_1$ has ordered list of operations as $\langle[A \gets 200]\rangle$ and $T_2$ has $\langle[A \gets -100, B \gets 100]\rangle$.
>
>Assume that both $A$ and $B$ start with an empty deposit, and the order of which these transactions are actually *issued* by the user is always $T_1$ and then $T_2$.
>
>We consider 4 systems that can execute these transactions. All of our systems can execute differently under different conditions, but there certainly is an *expected* execution, which ends up with 100$ in $A$ and 100$ in $B$.
>
>**System 1: Non-Atomic, Non-Serializable**
>A possible execution of this system would be that the first operations of $T_2$ executes first, fails, the transaction terminates and then $T_1$ executes without a problem.
>Now:
>- There is a brief period where another transaction can read $A = -100$ (right after $T_2$ fails and before $T_1$ finishes).
>- The end result is $A = 100, B = 0$.
>
>  In total:
>- There are 9 possible executions, depending on whether or not an individual operation in a transaction fails or not, and which one starts first.
>- There are 6 possible states in the end.
>
>**System 2: Atomic, Non-Serializable**
>A possible execution would be $T_2$ AND $T_1$ start at the same time, but $T_2$ fails when doing its second operation.
>- Depending how the transaction is being done, a brief period where $A=-100$ can be seen by $T_1$ (this is right after $T_2$ finishes its first operation but fails the second).
>- The end result is always, regardless of the implementation of the atomic transactions, $A=200$ and $B=0$, as if none of $T_2$ ever actually executed (in particular, $T_2$ will either *rollback* its first operation, which essentially converts $A=100$ to $A=200$, or it keeps the execution in memory and never actually reveal it till the end).
>
>In total:
>- There are 6 possible executions
>- There are 4 possible end states (makes sense, there are 2 transactions, each may fail completely or not, so $2^2 = 4$; Note that this example won't end up differently if the order of transactions were changed, since elementary operations are *commutative*)
>
>**System 3: Atomic, Serializable**
>A possible execution would be exactly as the previous one, however, when $T_1$ starts, **it will not see the intermediate result of $T_2$**, so it will end up with $A=200$ the moment that it finishes and the end result is the same. At no point will a transaction other than $T_2$ see a negative value for $A$.
>
>In total:
>- There are 5 executions (in particular, the execution which allows part of $T_1$ to execute between the two operations of $T_2$ is no longer possible)
>- 4 end states, as expected
>
>**System 4: Atomic, Strictly Serializable**
>There are exactly 3 possible executions, based *only* on whether or not $T_1$ or $T_2$ fail in the chain; Compared to the previous system:
>- The execution that allows $T_1$ to fail but $T_2$ to attempt is not allowed.
>- The execution that allows $T_2$ to finish first and $T_1$ later is not allowed.
>
>There is also, only 3 possible end states! (the same as the number of executions!)
>In particular, the number of possible final states is upper bounded by the number of possible executions; the bound being tight if executions result in distinct state values.
## Distributed Transactions

In its simplest form, transactions are done by getting a transaction ID from a master server (the `Tx ID`). Then the `Tx ID` is used to lock objects when reading/writing them.
- If commit finishes, then all if well.
- If commit aborts, all of the results are rolled back by getting rid of any update labeled with the above `Tx ID`.
- Almost always, a client has a cache, which keeps objects that were read for the transaction. If the server locks the objects, then it is easy to just execute all writes on the cache, and then push the results to the actual server.
  Another way is *Optimistic Locking*, where the data is read once into the cache with a version number. Before pushing the data, the object is locked for a moment (before this, it was free), and the version is checked. If the version is the same as before, commit is done and the lock is freed. If the version is changed, an error is returned to the client to retry and the lock is freed.

>[!FAQ] Why Use Optimistic Locking?
>It has the benefit of letting read operations continue when a transaction is going on. This will not violate serializability, but means that each write has a higher chance of failure due to concurrent access.
>
>This is the case for example, in RAMCloud:
>
>![[Pasted image 20241023194141.png|500]]

>[!IMPORTANT] Do Not Confuse Serializability with Linearizability
>Linearizability (which is what RAMCloud gives you), is a guarantee of operation *order* for a series of single-object updates.
>An update here is usually just read/write/delete, which is again exactly what RAMCloud gives you. This also checks out with the definition we had (i.e. an operation finishes sometime between its invocation and return to caller).
>
>Serializability on the other hand spans across all transactions that will ever commit on a system, even if they modify multiple objects.
>Essentially, you can have many read and writes done under the same transaction, and be confident that it is not getting interleaved with another transaction.

We have seen how to achieve linearizability, but Strict Serializability is new, how do you do that?
One way for that is Two Phase Commit! (2PC) Essentially, a system that implements 2PC would do the following:

- A client gets a transaction number from some coordinator.
- A client sequentially reports its operations for that transaction to the system, and the system will execute them.
  To prevent other transactions from racing with this, we should acquire read/write locks on each objects that we touch.
- Some protocol (e.g. 2PC) is then invoked with this to make results persistent across other replicas, while being correct under failures.

Again, RAMCloud for example didn't use any real lock like the above (it was optimistic locking all the way); Once the transaction finishes or aborts, all locks are released, letting the objects be available to other transactions.
## Plain Two-Phase Commit (Presume Nothing, `PrN`)

A topology for this protocol would include a series of *Cohort* servers, which could in theory be a whole  primary-backup system that looks like a single node. One cohort assumes the role of a *coordinator*, the first thing that kicks off the transaction (likely facing the client requests).

A coordinator always marks the first operation of a transaction. If a cohort dies, comes back after a transaction is started, and sees the commit mid-way, then it would immediately abort it when it notices that it has never seen the transaction start.

- A coordinator sends `REPARE` to all cohorts (Note that the client must have already contacted all cohorts about the transaction!).
- A cohort responds with a `COMMIT-VOTE` or `ABORT-VOTE`. The results is made persistent on the cohort.
- The coordinator sends the outcome, either `COMMIT` if everyone is ready, or `ABORT` if anyone rejected. The result is again made durable.
- The cohorts will then ACK the outcome.

### Cohort Details

Of course, if a Cohort dies in the middle of a transaction, comes back, and sees that the transaction is on-going, it MUST vote abort to the transaction.
This is needed since:
- We may not be prepared for the commit (i.e. we have not actually executed the transaction!), so we should abort so we don't miss updates.
- Serializability might be violated. When we return, all of our locks are free, so if a client comes in and modifies data on us, we'll get into a conflict with an on-going transaction!

Now, to protect against this (without logging) we have 2 things:

- The client always **marks** the first operation of a transaction. Seeing that, the cohort would be notified implicitly that a new transaction has been started without having to suffer an extra RTT for just the notification.
  If we get some operation for a transaction that we have not seen (e.g. if we die, miss the transaction's first operation, and get one of the others), we MUST vote abort when the coordinator asks us to prepare.

But this alone is not enough!
If a cohort gets the transaction notification, dies, comes back missing all the operations in between and picks them up in the middle, we are still toast!
So to prevent this, we actually *count* operations that each cohort does, and we also ask the client to keep track of the number of operations that it actually sent to the cohorts.

- When the client requests the commit, the coordinator actually embeds these message counts into the `PREPARE` message that it sends to cohorts, and a cohort MUST vote for aborting the transaction if the number of client-side operations does not match the number of operations that it actually executed.

These two combined, protect us against missing updates or not being Serializable *before* the transaction on 2PC starts.

For the rest, we need to talk about disks.
Persisting into disk is referred to as a *Forced-Write*. The distinction is made because usually we would use a disk for this, and disk don't actually ACK writes immediately, they would buffer it and then ACK it later. A forced write means that the (disk) ACK **MUST** be collected before doing the operation of the next steps.

Now, if a cohort dies while 2PC is ongoing, there are two cases:
- **We have no memory of the transaction outcome (i.e. Abort/Commit) after we come back**. In that case, we must ask the coordinator. <u>We will not serve clients until we get an answer! (so 2PC can be blocking)</u>
- **We still remember what the transaction outcome was.** Good! Carry on!

>[!IMPORTANT] What the Cohort must remember.
>A cohort MUST remember that it was prepared!
>To this end, it must force write to the log before voting after getting the `PREPARE` message.
>Similarly, before ACKing the `COMMIT` or `ABORT` message, the cohort must also log the outcome. This gives the coordinator the guarantee that the cohort will not ask the coordinator again about what the outcome was if it gives the ACK for the outcome.
>
>When a coordinator gets the outcome ACK from all cohorts, it can then just discard the ephemeral outcome value without a problem.
>Similarly, while the logging before ACKing on the cohort is necessary, there is no need to ACK immediately. We can group log writes and then delay ACKs and send them at once if it helps!

Thus, in `PrN`, we MUST make the outcome durable on the coordinator before sending any commit/abort message to cohorts.

### Coordinator Details

In the above, we referenced some "ephemeral outcome value", so presumably some data structure living in memory, right?
Indeed, the data structure for ongoing 2PC processes are held in a local database withing the coordinator which we call the *Protocol Database*.
This database is strictly within memory, but of course, some logging is needed since the coordinator can also crash!

The database itself, contains data like the following:

![[Pasted image 20241023223353.png|500]]

- `TID`: Transaction ID, which uniquely identifies the entry, and is inserted the moment a client informs the coordinator of the commit with 2PC.
- `STABLE`: The transaction start has been logged, we will not forget the existence of this transaction (although it does not reflect the outcome at all).
  Note that the log write for this is for now, **NOT FORCED!**
- `STATE`: The general state of the transaction according to 2PC:
	- `INITITATED`: The transaction is known to exist and a 2PC cycle has started.
	- `PREPARING`: At least one `PREPARE` message has been emitted.
	- `ABORTED`/`COMMITTED`: The outcome of the transaction has been logged and known.
- `{CID, VOTE, ACK}`: A set of entries, containing the ID of a cohort, its vote after receiving `PREPARE` and whether or not it actually ACKed the outcome.

The set of cohort states can be lazily garbage collected. The moment that `ACK` is true for an entry, we can safely drop the instance from the current cycle. When all instances are dropped, the whole commit entry can be garbage collected.

>[!IMPORTANT] What Coordinator Must Remember.
>Aside from non-forced logging of the existence of the transaction, the coordinator for `PrN` usually needs to remember 2 things.
>- Before sending the commit outcome, the Coordinator MUST force write to the log the actual outcome of the value. Once this write is finished, the fate of the transaction has been sealed. Eventually, all nodes will consistently know what to report for this transaction.
>- After all ACKs come, the Coordinator can write an `END` record in the log for the transaction. **This one is NOT forced**, since all it does is prevent unnecessary database entry reconstruction of commits that we are done with.


In brief:

![[Pasted image 20241023230010.png|600]]

This `PrN` needs 2 forced log writes on each cohort, and 1 forced and 1 non-forced log write on the coordinator.

As you see above, we only need to start logging when the outcome is known. This means that even if everyone votes to `COMMIT` but we crash, the transaction ends up in an unknown state.

This is where we should start *presuming* about transaction outcomes.
In particular, a transaction is presumed to have aborted in `PrN`. Thus if a cohort at any point asks about the transaction outcome (for example when itself crashes after logging the transaction being prepared), the coordinator will know that transaction *exists* but it will have no idea who voted for it and when, thus, it *assumes* that the transaction was aborted.

The client will eventually know of this (even though the whole transaction might be perfectly fine to commit really), and will have to retry.
This is where optimization becomes a thing. In particular, **what should the coordinator do when it is asked about the outcome of a transaction that it knows exists, but does not know what happened to it?**
### Optimization: Presumed Abort (`PrA`)

Due to how the protocol is designed, if a transaction is not completed when we recover from a crash, we can be sure that it was aborted. As such, there is no reason to make aborted transactions persistent in the log, only committed transactions need to be persisted in the log.

Now, `PrN` already assumed that a transaction with an unknown outcome is aborted, but it *still logged the outcome of valid `ABORT`s nonetheless!* This is wasteful, since the moment the transaction aborts, we can just chuck the database entry into the bin and forget about it!

This saves a forced log write for all transactions that abort. It also means that aborted transactions need not be ACKed at all.

![[Pasted image 20241023230329.png]]

### Optimization: Presumed Commit (`PrC`)

Similarly, we can change the protocol in such a way so that logs are only finished for transactions that actually commit. We force a write for the initiation of each commit (this was previously non-forced, and even optional, but now we MUST do it), and another forced write for actually committing a transaction.
When a transaction aborts, we end the log entry for the transaction.

The good thing is that in this model we don't need to get ACKs for commits in the second phase. As long as the outcome is logged, we are safe. The price is that a forced initiation record is required for ALL transactions (even those that will abort).

![[Pasted image 20241023230641.png]]


#### The Case Of Read-Only Transactions

For a given transaction, a cohort can be read-only, meaning that it is previously known from the transaction preamble that the client sends that a transaction never modifies data.
This can be exploited for more optimizations, in particular, *such cohorts need not log anything and need not know the outcome of the transaction*.

To this end, we add a `READ-ONLY-VOTE` message for responding to `PREPARE`, which informs the coordinator that there is no need to collect ACKs or even send the outcome to the cohort.
When all cohorts are read-only, the coordinator need not send any outcome message.

So `PrA` and `PrN` need not log anything for read-only transactions, however, `PrC` is a different story. It needs to treat these transactions *exactly* like any other, so it needs 2 log writes (initiation and outcome logs), but the outcome log need not be forced (note that the client that issued the transaction already either knows the read values or will know that it has to retry!)

## A New Presumed Commit (`NPrC`)

The initiation record of `PrC` is such a stinger (it has to be done for *all* transactions!), can we throw it away perhaps? Yes! With some cunning plans ...

Assume monotonically increasing transaction identifiers and maintain the set of in-flight transactions. To do that, create the following sets:

- `COMM`: Set of `Tx ID`s for transactions that we know are committed (we know this from the logged outcome in the log)
- `TID_l`: The smallest `TID` for a transaction without an outcome.
- `TID_h`: The largest `TID` for a transaction that was initiated

The idea is that after a coordinator crash, we can:
- Assume transactions less than or equal to `TID_l` are committed.
- Assume transactions strictly larger than `TID_l` have been aborted.
- For transactions that we presume to have aborted, we can look at `COMM` to shrink the set.

![[Pasted image 20241023232743.png|600]]

The set of IDs between `TID_l` and `TID_h` that are not in `COMM`, are in-flight commits. If we crash and recover, the set of IDs:
$$
\text{IN} := [\text{TID}_l \; .. \; \text{TID}_h]\; - \;\text{COMM}
$$
Can be safely presumed to have been aborted, and we can then bulk-initiate a transaction for all of them at once after recovery.

In the above:
$$
[\text{TID}_l\;..\;\text{TID}_h]\; := \{t\;|\;\text{TID}_l < t < \text{TID}_h\}
$$

Note that all cohorts need to log the commit preparation durably, since if they don't they wouldn't know if they did or did not apply the commit when they crash. Remember that the coordinator only lets you know if the commit wasn't aborted, it will not tell you if you actually applied it!

>[!NOTE] Coordinator Behavior After Crash With `NPrC`
>The coordinator will start with a blank protocol database. It reads its log, and finds `TID_l` and `TID_h`. The entire range of $[TID_l\;..\;TID_h]$ is then reconstructed into the database.
>- For any transaction that has the state of `COMMIT`, send the `COMMIT` outcome and purge the entry .
>- For any other transaction, assume abort, send `ABORT`, **COLLECT ACKs** and then purge the entry.

Afterwards, we need to bump up `TID_l` and `TID_h`, it is not safe to reuse that range. This can be done either explicitly (i.e. move `TID_l` somewhere after `TID_h` and put `TID_h` right next to it), or we can shift them by a pre-defined delta.

![[Pasted image 20241015132951.png|500]]

The protocol itself messages in exactly the same way as `PrC` would, but the difference is with what we write in the logs. For read-only transactions, we can be as lazy as we want to be.
