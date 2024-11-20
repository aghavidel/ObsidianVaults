**Paper:** [Practical Byzantine Fault Tolerance]([bfs.pdf](https://www.scs.stanford.edu/nyu/03sp/sched/bfs.pdf))

This note is in reference to the grand-daddy of pretty much every Byzantine fault tolerant system ever.

The goal is to implement a *practical* algorithm (one that is not heavy in computation and reasonably live) that can withstand malicious nodes (i.e. Byzantine nodes). In the above, computation mainly refers to message integrity verification (i.e. digital signatures, or hashing).

The promise of the algorithm is that it provides safety and liveness for any state machine replication algorithm in a network of $n$ nodes, provided that at any point of time, at most $\lfloor \frac{n-1}{3} \rfloor$ of the topology is faulty (faulty meaning crashed and/or malicious).

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

The algorithm is carried out throughout rounds that we call *views*. Each view is indexed with a number. We start from view 0 by default.

Let $\mathcal{R}$ be the set of replicas. The algorithm requires that $|\mathcal{R}| > 3f$. There is no reason to go above $3f+1$, since it would just increase communication overhead, so we just let the number of replicas be exactly $n=3f+1$. 
In this case, we can index each replica as $0,1,2, ..., 3f$. We use this to define the *primary* node of a view $v$ as the node index $v\mod{n}$.

Thus, as views change, so do replicas, and it is also possible that a faulty node actually becomes a replica. As we will see, if this node attempts to do anything harmful, the view will change and we come up with a new replica as the primary.

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
For some primary node $p \neq i$.
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

This means that for a given view, and sequence number, no two messages with the different digests can be prepared for commit at the same time. This means that messages with different digests within the same view $v$, MUST be assigned different values of $n$, and this implies a total ordering on them as long as $n$ increases monotonically.

>[!EXAMPLE] Proof Of Invariant 1
>Assume the contrary, so there must exist some message with a different digest that is also prepared at the same time as $m$. Noting the definition for $\text{prepared}$ implies that:
>
>- Both messages are in the log for some nodes 
>- Both messages have been pre-prepared
>- Both messages have a match set at least as large as $2f$
>
> These imply a total of $2f+1$ distinct nodes (the primary and $2f$ backups) have a `PRE-PREPARE` or `PREPARE` with the same values of $v$ and $n$ in their logs. 
> 
> We have at 2 prepared messages, thus a total of $2(2f+1)=4f+2$ log entries exist with the same $v$ and $n$, and by pigeon-holing it is obvious that there must be $f+1$ distinct nodes that have **TWO** such entries in their logs. At least one of these nodes is not faulty, and thus we have a contradiction, because if this node is not faulty, it should have rejected one of these entries because they have the same $v$ and $n$, but the digests are different.

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
>\text{committed-local}(m, v, n, i) \implies committed(m, v, n)
>$$

>[!EXAMPLE] Proof of Invariant 2
>This is simple. The match set for `COMMIT` messages being at least $2f+1$ implies that at least $2f+1$ nodes must satisfy $\text{prepared}$.
>
>Of these, at least $f+1$ are non-faulty. This set of nodes is our $I$ in the definition for $\text{committed}$ and thus $\text{committed}$ is trivially satisfied from its definition.

