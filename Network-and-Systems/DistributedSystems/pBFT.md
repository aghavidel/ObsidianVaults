**Paper:** [Practical Byzantine Fault Tolerance]([bfs.pdf](https://www.scs.stanford.edu/nyu/03sp/sched/bfs.pdf))

This note is in reference to the grand-daddy of pretty much every Byzantine fault tolerant system ever.

The goal is to implement a *practical* algorithm (one that is not heavy in computation and reasonably live) that can withstand malicious nodes (i.e. Byzantine nodes). In the above, computation mainly refers to message integrity verification (i.e. digital signatures, or hashing).

The promise of the algorithm is that it provides safety and liveness for any state machine replication algorithm in a network of $n$ nodes, provided that at any point of time, at most $\lfloor \frac{n-1}{3} \rfloor$ of the topology is faulty (faulty meaning malicious, not crashing).

We'll describe the exact meaning of the safety condition, and what it entails later.

# Introduction

We want to implement state machine replication in the presence of byzantine faults with asynchronous networking. This means that:
- Network can arbitrarily delay messages and do all sorts of horrible stuff to them (but the delay is the main sticking point, TCP can handle the rest)
- Faulty nodes can also behave arbitrarily, meaning that they can:
	- Refuse to answer certain messages
	- Send gibberish
	- Be malicious
	- Attack other nodes by slowing them down

To combat this model, we are going to make some reasonable assumptions:
- **Failures are independent**. This means that taking control or compromising does not immediately generalize to other nodes. For example, if all nodes use Linux, and some security vulnerability is found in the current kernel that allows Remote Code Execution, then this immediately generalizes to all nodes, meaning that every node is faulty to some extent. 
  We will *not* consider these cases. There is no fault tolerant algorithm that can survive such a state and so we assume that measures are taken to make sure the above does not happen (for example, OS versions are each slightly different, or completely different but provide the same interface, etc.)
- **Efficient cryptographic functions exist on each node.** Meaning that we assume some reasonably powerful CPU exists on each node that can handle cryptographic functions. These include generating message digests, signing, and MACs.

>[!REMINDER] How Pub-Key Signatures Work
>This is simplified, but:
>- All nodes scream their public keys at others (there is no sense lying here, because if you lie about your public key, you can't send messages to people without them flagging your messages as broken)
>- Each message that we want to sign, like $m$, we first encrypt a digest of it (or itself, if you have no care for performance!) and then we send it with our message (encryption is done with the private key).
>- The receiver will decrypt the result with the associated public key and hash the received message if needed. If they agree, it declares that the message is OK.

In this paper, a message $m$ will have its SHA-1 digest noted as $D(m)$, and the signed version of $m$ by a sender node $i$ is denoted as $\langle m \rangle_{\sigma_i}$. In practice, the simplest implementation of this would be:
$$
\langle m \rangle_{\sigma_i} = \langle m, E(D(m), k^{(i)}_{private}) \rangle
$$
Where $E(s, k)$ would be the encryption of string $s$ with a given key $k$.

So here are the things that an adversary **CAN** do:
- Delay non-faulty nodes by some finite time.
- Instantly know when messages are being exchanged and can slow down or cause them to get corrupted.
- Exchange arbitrary messages with other faulty nodes or just bother non-faulty nodes with gibberish or non-gibberish.

What we assume they **CAN'T** do:
- Fake a signature as if it came from another node
- Compute any part of the original information from its digest
- Look for hash collisions given a message $m$

## System Properties

The safety and liveness properties that the system provides are:
- **Safety**: Operations provide Linearizability. Meaning that they will not allow effects to be externalized before the client that issued it returns, as long as no more than $\lfloor \frac{n-1}{3} \rfloor$ replicas are faulty.
- **Liveness:** All clients eventually see the result of their operations assuming that no more than $\lfloor \frac{n-1}{3} \rfloor$ of replicas are faulty and the network provides at least weak synchrony.

>[!REMINDER] Liveness In Asynchronous Networks
>Remember the FLP result that no consensus algorithm in a 3 node network can exist in an asynchronous network with one faulty replica. As such, if safety is to be provided assuming full asynchrony, then liveness MUST require some sort of synchrony, otherwise pBFT could be used to implement a consensus protocol on an asynchronous network with 3 nodes and 1 faulty node.

>[!FAQ] Weak Synchrony
>Weak synchrony could mean a lot of things in different contexts, but in our case, it means the following:
>
>Let $delay(t)$ be the time between receiving a message at its destination and the time point $t$, when it is actually emitted from the source. Weak synchrony asserts that $delay(t)$ can only grow faster than $t$ for some finite amount of time.
>The immediate result of the above is that there is a point $t^*$ for any $t$ that we would have $t^* = delay(t) + t$, where we will finally receive the message. This requires that the sender retransmits messages though, since they can get outright dropped.

The resiliency of this system is optimal, since as we saw in [[BFT]], $3f+1$ is the minimum number of replicas needed to provide asynchronous safety and weakly synchronous liveness. 

For our case, this makes sense, because at any point, $f$ replicas can crash or refuse to respond, so the algorithm must be able to decide with only $n-f$ replicas at any point. However, because the network is only weakly synchronous, the nodes that appear crashed may only have been heavily delayed (either because of an adversarial network, or some coordinated attack by the faulty nodes), and as such we cannot assume that all of the nodes that did respond are non-faulty.

To protect against this, non-faulty nodes must always outnumber faulty ones in the remaining $n-f$ set of nodes, thus:
$$
(n - f) - f > f \Rightarrow n > 3f
$$
We will see that the above is good enough to provide linearizability, but not good enough to be safe against faulty *clients*. Indeed, a faulty client can exchange any valid request with gibberish and wreak havoc. **This part needs to be handled separately, pBFT will NOT provide any protection against that**.

# Design

The algorithm is carried out through rounds that we call *views*. Each view is indexed with a number. We start from view 0 by default.

Let $\mathcal{R}$ be the set of replicas. The algorithm requires that $|\mathcal{R}| > 3f$. There is no reason to go above $3f+1$, since it would just increase communication overhead, so we just let the number of replicas be exactly $n=3f+1$. 
In this case, we can index each replica as $0,1,2, ..., 3f$. We use this to define the *primary* node of a view $v$ as the node index $v\mod{n}$.

***Take note of the fact that the primary of each view is deterministic! (this is crucial in proving view-change protocol correctness ...)***

Thus, as views change, so do primaries, and it is also possible that a faulty node actually becomes a primary. As we will see, if this node attempts to do anything harmful, the view will change and we come up with a new replica as the primary.

The general workflow of the algorithm when nothing bad is happening is like the following:
- An authenticated client issues a request, which invokes some service on the current primary node.
- The primary node multicasts the request to all other replicas
- Each replica executes the request (or does something else if it is faulty or expects the primary to be faulty ...) and **sends the reply to the CLIENT (NOT the primary)**
- The client waits for at least $f+1$ replies from different replicas that have the same result. This value is declared as the result of the operation.

## Client Operation

A client $c$ requests the execution of some operation $o$ by issuing a message formatted as $\langle \text{REQUEST}, o, t, c \rangle_{\sigma_c}$. As you see, client messages are also signed, which requires the cluster to know of the public key of the client, which should be done at least as part of the authentication procedure.

In the above, $c$ is client identifier (it can be as simple as an index), and $t$ is a timestamp. This timestamp need only order requests that are coming from $c$ itself, and as such, a steady, local wall-clock would be enough for this.

The client then awaits reply messages from replicas. These messages have the form $\langle \text{REPLY}, v, t, c, i, r \rangle_{\sigma_i}$, where $v$ is the view number, $i$ is the replica index and $r$ is the result of the operation. The values $t$ and $c$ must be kept consistent with their values in the original request message that the client issued, if not, the client should not accept them.

The client waits, and as such, it needs to have a timeout. Once this timeout expires, it resends the request, but NOT just to the primary, it sends it to all replicas and awaits responses. 

- If a request has been completed, the replica can return the result directly. This requires that each replica **remembers the result of the last operation that a client issues**. This can be done since each replica must at least know the public key of the client, so it can index requests per public key in the worst case.
- If a replica receives a new request, but it has no idea what the result is, it must then forward it to the primary. The primary treats this request as a brand new one and multicasts it to others (again with the constraint that no one will re-execute requests if it remembers the last one).
  If this gets delayed, other replicas will flag the primary as faulty and change the view.

For now, assume a client issues requests one at a time. A client returns and moves to the next operation if and only if it receives at least $f+1$ matching replies.

>[!FAQ] $f+1$ Is Optimal
>We have $f$ faulty replicas at most. As we discussed above, we assume they can be really strong, and as such, they can instantly know that a client submitted a request. Thus, a scenario here would be for them to immediately tell the client that the operation is done, and this would immediately violate linearizability (if the client reads back the value that it wrote next, it can get garbage). Thus, $f+1$ is the minimum number of answers that you need to make this work.
## Primary Operation

The primary walks the replicas through 3 stages; *pre-prepare*, *prepare* and *commit*. 

### Pre-Prepare

The primary picks a sequence number $n$ and assigns it to the request that it gets from the client (we will from now on refer to this request in its entirety as $m$). It will then emit a message of the form:
$$
\langle \langle\text{PRE-PREPARE}, v, n, d \rangle_{\sigma_p}, m \rangle
$$
Where $d$ is the digest of $m$. This gets multicast to all replicas and once that is done, the primary appends this message to its log. 
The goal of `PRE-PREPARE` is make sure that the cluster knows that the message $m$ (or more precisely, the digest of it) was seen in view $v$ by the primary and given a sequence number $n$, which must be monotonically increasing.
This will be important when we discuss view changes.

>[!NOTE]
>While the above makes it look as if the `PRE-PREPARE` packet and the message $m$ are completely glued together, there is no need for them to be. The messages need only be piggybacked. However, if we decide to separate the transport for these two (which the paper seems to hint that we do), then we should be able to cope with out of order delivery (i.e. receiving $m$ first and the `PRE-PREPARE` later).

>[!IMPORTANT] Conditions For Accepting A `PRE-PREPARE` message
>The simple ones would be:
>
>- The signatures for both the client request and the `PRE-PREPARE` must be valid.
>- The digest $d$ must be correct.
>- The receiver is in the same view $v$ that it sees in the packet.
>- No previous `PRE-PREPARE` was seen with the same $v$ and $n$ that contained a different digest.
>  
>  But this is not enough. A faulty replica can pick a very high value for $n$, skipping a huge range of $n$s to create a likely collision later. To this end, we introduce another constraint:
>  
>  - We must have $h < n < H$ for some values $h$ and $H$, which we shall discuss later.

Let $L_i$ be the log of node $i$, we define the predicate $\text{pre-prepared}$ as:
$$
\text{pre-prepared(m, v, n, i)} \iff \langle \text{PRE-PREPARE}, v, n, D(m) \rangle_{\sigma_p} \in L_i
$$
For some primary node $i \neq p$.
### Prepare

If a `PRE-PREPARE` is accepted, we say that the replica is in *Prepare* phase and it announces this by screaming $\langle \text{PREPARE}, v, n, d, i \rangle_{\sigma_i}$. Both this `PREPARE` message and the entire `PRE-PREPARE` message are appended to the log. If the `PRE-PREPARE` was rejected, we do nothing.

The conditions for accepting `PREPARE` messages are the same as `PRE-PREPARE`. We also say that a `PREPARE` and a `PRE-PREPARE` message *match*, when their corresponding values for $v, n$ and $d$ match. Let $\text{match}_i(\text{PREPARE}, m, v, n)$ be the set of `PREPARE` messages in $L_i$ that share the same values for $v, n$ and the digest $d$ for message $m$. Note that `PREPARE` messages are only emitted by backups, so the primary node does not count for anything in $\text{match}$.

We define the predicate $\text{prepared}$ with the following:
$$
\begin{align*}
	\text{prepared}(m, v, n, i) \iff 
		&\wedge m \in L_i \\
		&\wedge \text{pre-prepared}(m, v, n, i) \\
		&\wedge |\text{match}_i(\text{PREPARE}, m, v, n)| \geq 2f
\end{align*}
$$
The above definition gives us a nice invariant. In particular:

>[!IMPORTANT] Invariant 1
>$$
> \begin{align}
>	&\forall m, m' &\;s.t.\quad & D(m) \neq D(m'),\\
>	&\forall i, j \in \mathcal{R} &\;s.t.\quad & \text{$i$ and $j$ are not faulty},\\
>	&\quad &\quad  &\text{prepared}(m, v, n, i) \Rightarrow \;!\;\text{prepared}(m', v, n, j)
> \end{align}
> $$

This means that for a given view, and sequence number, no two messages with different digests can be prepared for commit at the same time. This means that messages with different digests within the same view $v$, MUST be assigned different values of $n$, and this implies a total ordering on them as long as $n$ increases monotonically.

>[!EXAMPLE] Proof Of Invariant 1
>Assume the contrary, so there must exist some message with a different digest that is also prepared at the same time as $m$. Noting the definition for $\text{prepared}$ implies that:
>
>- Both messages are in the log for some nodes 
>- Both messages have been pre-prepared
>- Both messages have a match set at least as large as $2f$
>
> These imply at least a total of $2f+1$ distinct nodes (the primary and $2f$ backups) have a `PRE-PREPARE` or `PREPARE` with the same values of $v$ and $n$ in their logs.
> 
> We have at least 2 prepared messages, thus a total of $2(2f+1)=4f+2$ log entries exist with the same $v$ and $n$, and by pigeon-holing it is obvious that there must be $f+1$ distinct nodes that have **TWO** such entries in their logs. At least one of these nodes is not faulty, and thus we have a contradiction, because if this node is not faulty, it should have rejected one of these entries because they have the same $v$ and $n$, but the digests are different.

>[!REMARK]
>If the digests of two messages collide, the above no longer implies total ordering between differing messages, but the probability of that is very small ...
### Commit

Once $\text{prepared}(m, v, n, i)$ is true in a node $i$, it moves on to the `COMMIT` phase by screaming $\langle \text{COMMIT}, v, n, d, i\rangle_{\sigma_i}$. Once a node $i$ receives $2f+1$ agreeing commit messages (including its own if possible!), then it applies the operation in the client request to its state machine. The above condition can also be written as:
$$
\text{committed-local}(m, v, n, i) := |\text{match}_i(\text{COMMIT}, m, v, n)| \geq 2f+1
$$
Once the request is applied to the state machine, a replica replies directly to the client with the result of its operation. 

So why the above works? We define the following:
$$
\begin{aligned}
	\text{committed}(m, v, n) \iff \exists\; I \subset \mathcal{R} \;\text{s.t.}\; &\wedge\;|I| \geq f+1\\ 
		&\wedge\;\forall i\in I: 
			&&\wedge\;\text{$i$ is not faulty} \\
			&&&\wedge\;\text{prepared}(m, v, n, i)
\end{aligned}
$$
So essentially, when $\text{committed}(m, v, n)$ is true, a majority of non-faulty nodes are prepared. Following from invariant 1, it means that a majority of non-faulty nodes have agreed upon a total ordering for requests and as such, committing a request with such properties would be safe.

>[!IMPORTANT] Invariant 2
>For any non-faulty node $i$:
>$$
>\text{committed-local}(m, v, n, i) \implies \text{committed}(m, v, n)
>$$

>[!EXAMPLE] Proof of Invariant 2
>This is simple. The match set for `COMMIT` messages being at least $2f+1$ implies that at least $2f+1$ nodes must satisfy $\text{prepared}$.
>
>Of these, at least $f+1$ are non-faulty. This set of nodes is our $I$ in the definition for $\text{committed}$ and thus $\text{committed}$ is trivially satisfied from its definition.

If message delivery is fair (which is a requirement, otherwise, a network can eternally drop messages and we would not be able to progress), then *eventually*, $\text{prepared}$ becomes true, and from invariant 1 it follows that total ordering will be achieved.

All $\text{prepared}$ nodes will eventually scream `COMMIT`, and as such, with the same fairness conditions, $\text{committed-local}$ will become true. From invariant 2 it follows that at this point, $\text{committed}$ also becomes true, which means that at least $f+1$ non-faulty nodes will respond to the client with agreeing messages. At this point, the client returns and as such, liveness follows from fairness of message delivery.

**Unless**, there is a view change, in which case liveness might be jeopardized by constantly changing views ... so it's time to discuss that.

## View Change

All the discussions above assume view change is delayed long enough so that we can commit and let the client move forward. Otherwise, the final step where we can be *sure* that at least $f+1$ nodes will respond with agreeing values is no longer a guarantee.

However, we also now uncover a pretty concerning problem.
If a (non-faulty) replica dies, returns and wants to resume operation, then it must be updated with the current state of the committed log. This requires nodes to share with it what the current state is.
So far so good, until we realize that:
- Repeatedly screaming a gigantic log into the network, puts a lot of overhead on our communication. This makes sure that the system slows down more and more as we progress.
- Repeatedly *compressing* a gigantic also becomes prohibitive.

For this, note that a log entry becomes entrenched when it gets committed. The moment we are sure that at least a majority of non-faulty nodes (i.e. $f+1$ non-faulty nodes) have committed an entry, we can garbage collect it.
However, we need to keep some *proof* about doing this, some signature that can be verified by another replica that we did in fact do this. Otherwise, we are back to square one and would have to send the gigantic log at some point.

Thus, we need two things:
- A good way to checkpoint a log and compress it by chopping the tail.
- A good way to prove that we did it and we are not lying.

The second point becomes crucial in two particular scenarios:
- If a non-faulty replica dies, misses a bunch of updates, and returns to see that state of the log is quite different, it needs proofs from others before it is able to safely accept updates from others.
  If a faulty replica manages to somehow influence this process, then it is very easy to see how Linearizability can be compromised if enough faulty replicas die and comeback.
- When a view changes, and suddenly an on-going update looks like it happened in a previous view instead of the current one, we need to agree on when it happened. Otherwise, some non-faulty replicas will refuse to accept it, while faulty replicas can do it instead, and we get another safety violation again.

So, we need checkpointing.

### Checkpointing

We introduce another message type, `CHECKPOINT`, of the form $\langle \text{CHECKPOINT}, n, d_s, i \rangle_{\sigma_i}$. Now, we need some extra semantics before we discuss what this means.

- Once a request is received (a request being a message $m$ from the client), it is executed, which generates a *state*, and a response which we return to the client. The state is something that is in essence, a digest of the log in and out of itself, since it reflects what happens when we execute the log.
  The deterministic nature of the state machine that pBFT executes, also means that we can just set the state machine to this *state*, and we can resume operation afterwards as if nothing happened.
- A *state* can be identified from the sequence number of its log entry, $n$. We can also have a sequence of states that are generated by executing a series of contiguous messages from the log and use that instead.

A *checkpoint*, formally means the resulting state of executing a series of messages, and to prove that we have a checkpoint, we need only create a *digest* out of it and send that instead, and (assuming the receiver indeed actually has the associated states logged), then it can decide if we are lying or not.

That is what we reflect in the checkpoint message, the checkpoint contains $n$, the sequence number of the last message that the checkpoint encompasses and $d_s$, the digest of the state **up until** then. This means that we need to be able to create collision-resistant hashes that can also be computed *incrementally*, otherwise, we need to hash the entire sequence of states that were generated by the log to get $d_s$.

Now, a faulty node can do the same, so again, we need to get a quorum of non-faulty nodes to agree about a checkpoint being generated before we can declare it stable, and flush the log up to and including the last message with sequence number $n$ (this includes any commit, prepare or pre-prepare messages, and we do not need to make this contiguous).

We say that a checkpoint is *stable*, if a node has collected $2f+1$ agreeing on the checkpoint as mentioned above (this again, may include its own). This set of $2f+1$ agreeing checkpoint messages are what **proves** that a checkpoint makes sense.
This set of messages is pushed to the log, and afterwards, a node can safely discard any earlier checkpoint and its proof messages.

>[!NOTE] Setting $h$ and $H$
>Remember these two? The low and high water marks for sequence numbers?
>Checkpointing is what advances these. When a node checkpoints something in a stable manner, we can move $h$ to $n$, and set $H$ to be some $n+k$ for some value of $k$.
>
>We must set $k$ to be reasonably large, since if we don't do that, we will spend to much time generating proofs for states rather than doing actual work.

### Moving On From A View

Now the main part, what to do in order to progress a view that is stuck.
The most obvious way that a view can get stuck is for its designated primary, i.e. the replica $r= v \mod n$ to die (or fake that it is dead, if it is Byzantine, or just become really slow and look like it is dead). This ensures that `PRE-PREPARE`s do not get emitted, and we are stuck. So, it is obvious that replicas need a timeout to decide when a primary is dead.

To this end, each replica maintains a timer. This timer:
- Starts whenever both the following hold:
	- It is not already running
	- The replica has received a valid request from a client (either by the designated primary, or from the client itself), and has not yet been able to commit it.
- It gets reset whenever:
	- A request gets committed, but some other request is still in queue
- It stops:
	- When a request gets committed and there are no further requests to process

>[!IMPORTANT] Heartbeats?
>Unlike Raft, or any other Byzantine Fault *intolerant* protocol, heartbeats are not the main measure for keeping liveness.
>In fact, principally, we do not need heartbeats at all for this protocol, the timer above is all we need for this purpose.
>
>There might be optimizations that can use heartbeats in some sense, but for deciding liveness for primaries, they are clearly a problem, since if a faulty node becomes a primary, we explicitly want to make sure we can move on if they don't do anything. For this purpose, heartbeats are meaningless, actual progress for client messages is the main thing we need.
>
>The downside of this of course, is the fact that clients need to propose to all nodes in the network, otherwise, we would not know when to start the timer. This shows you why we need all-proposing clients for this algorithm.

If on a replica $i$, the timer expires, then:
- $i$ stops accepting any message other than `CHECKPOINT`, and messages regarding view changes (i.e. `NEW-VIEW` and `VIEW-CHANGE`, which we will discuss shortly)
- It then screams out a view change message of the form $\langle \text{VIEW-CHANGE}, v+1, n, \mathcal{C}, \mathcal{P}, i \rangle_{\sigma_i}$, where:
	- $n$ is the sequence number of the last stable checkpoint and $\mathcal{C}$ is its proof of stability (a proof, again, is a set of $2f+1$ `CHECKPOINT` messages from different replicas with the same sequence number and agreeing digests).
	- $\mathcal{P}$ is a set (or sequence) of sets of messages $\mathcal{P}_m$, where:
		- $m$ is any prepared message at the node that is not within the checkpoint (which implies that they have sequence numbers strictly larger than $n$)
		- $\mathcal{P}_m$ contains at least, valid messages of:
			- The `PRE-PREPARE` for $m$ in view $v$, signed by the designated primary
			- $\text{match}_i (\text{PREPARE}, m, v, n)$, which must have a size of at least $2f$, otherwise, it would be obvious that we are lying about being prepared for $m$ and will be flagged as faulty.
			Note that each $\mathcal{P}_m$ maps uniquely to a sequence number. Thus, in some sense, $\mathcal{P}$ is a set of sequence numbers. Keep that in mind.

Now, here is the main thing. The primary of the next view, $v+1$, is again, pre-determined, However, it cannot initiate the view change when it just sees view change message from a replica or itself. It needs to again, collect at least $2f+1$ of them to make sure that a majority of them agree that we must change the view (in some sense, it is requesting votes from others to become leader); However, since the designated primary already implicitly agrees to change view by default, we can just say $2f$ messages from others, plus itself.

Once this condition is fulfilled, the designated primary screams a `NEW-VIEW` message of the form $\langle \text{NEW-VIEW}, v+1, \mathcal{V}, \mathcal{O} \rangle_{\sigma_i}$. Before we discuss what is actually in it, let us discuss what *should* be in it.

- It must contain a proof of view change, since we are collecting view change messages, proof would be the set of collected messages.
- It must contain a sort of *patch*, that makes sure that uncommitted messages from the previous view, are known in the new view.
  The smallest patch would be to make sure that all outstanding client messages have at least a pre-prepare message attached to them, because otherwise, every non-faulty node would drop any hint of a message that it knows it is not pre-prepared for.

Now, the contents of this message are:
- $\mathcal{V}$ is the proof for view change, a set of $2f$ `VIEW-CHANGE` messages from different replicas, and the unique `VIEW-CHANGE` message from itself (this one is special, since this is going to be the primary of next phase). There is no need for the `VIEW-CHANGE`of the designated primary to be sent already; if it wasn't, then $\mathcal{V}$ implicitly piggybacks it on the `NEW-VIEW`message.
- $\mathcal{O}$ is a set of pre-prepare messages, made with the following procedure:
	- The primary sifts through the checkpoints that it receives as part of the $2f+1$ view changes in $\mathcal{V}$, and chooses the sequence number of the latest stable checkpoint in that set, call it $\text{min}_s$.
	- The primary sifts through the prepare messages in $\mathcal{V}$ (which are part of its $\mathcal{P}$ component). The largest sequence number seen in this process is recorded as $\text{max}_s$.
	- The goal here, is to make a pre-prepare message for all of the messages that are outside of the checkpoint.
	  Thus, we iterate over all available sequence numbers in the interval $[ \text{min}_s, \text{max}_s ]$, and we create a brand new `PRE-PREPARE` for them. As we mentioned earlier, each $\mathcal{P}$ is essentially a set of sequence numbers, so for each sequence number, we can get the associated message digest for that sequence number in $\mathcal{P}$ in it and make the pre-prepare message.
		- In the particular case that no such message digests for a sequence number exists, we use a placeholder, a digest $d_{null}$ for a special *null* message. This message, once executed, does nothing (and as such, a hash collision with it matter not at all).

This message is screamed in the network. All non-faulty nodes that receive this will verify this, and if it is correct, officially enter the new view.
