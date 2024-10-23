# Byzantine Consensus

Until now, we discussed only within the context of fail-stop crashes.
This means that we only consider scenarios where a server would just stop working and that's it. There is a sense of honesty here, in the sense that as long the machine recovers from the crash correctly without losing critical state, our algorithm operates "correctly".

For this lecture, we will focus on *Byzantine Failures*, which we define as a case where a server:
- Can act maliciously, lying about its state or delaying/dropping messages to others.
- Crash arbitrarily

A consensus protocol that operates under the failure model of Byzantine nodes, is called a **Byzantine Consensus** protocol, which means that it is able to reach consensus, even in the presence of Byzantine nodes.

>[!NOTE] 
>In this day and age, Blockchains, Bitcoin in particular, have become the most widely used Byzantine Consensus protocol.
>Indeed, much that we learn today provides context for that protocol as well.

Let's consider the above a little bit more, a mode of operation in the above is a **Permissioned Blockchain**
In this mode, consensus is obtained over *a set of predefined* servers. As such, as long as that set of servers can survive a certain amount of Byzantine nodes, we would be safe.

>[!REMINDER]
>Until now, we considered 2 types of replication that would help:
>- **Primary-Backup**: With $f$ backups and 1 primary, we would be able to tolerate $f$ failures, so we need $f+1$ nodes in total.
>- **Consensus**: If we assume $f$ nodes can fail, then we must have at least $f+1$ nodes to remain available. However, we must make sure that consensus is always possible among the remaining nodes. To this end, we must make sure that in our $n$ nodes, even if only $n-f$ servers are available, we need at least a majority quorum to output. Thus we must have $n-f > f$, thus we need at least $n \geq 2f+1$.

For Byzantine failures, the case is different.
Again, we assume the worst case, only $n-f$ nodes would be available for service, but we must make way for the possibility that of these $n-f$, at most $f$ servers might have lied about their state (we will discuss why we can have such a scenario in the following discussions).

As such, over these $n-f$ nodes, we must have a majority quorum of honest nodes, even with $f$ lying nodes. Thus we must have:
$$
n -f \geq (f+1) + (f) = 2f+1 \Rightarrow n \geq 3f+1
$$
The first left side should be interpreted as "At least $f+1$ honest nodes remain, when at most $f$ nodes lie".

So, no matter what our protocol does, as long as we can have $f$ nodes that can arbitrarily slow messages down and do horrible things, we need at least $3f+1$ nodes in our system. 

# Practical Byzantine Fault Tolerance (PBFT)

This paper was rather ignored in the industry, before Bitcoin came and shook things up. With the benefit of hindsight, this paper has been revisited in the last decade with sharper eyes.

PBFT is built on the following assumptions:
- Broadcasts and Multicasts are feasible. We cannot assume what the set of working servers are at any point, so we would just scream everything at everyone, even if it is none of their business.
	- This has a downside, broadcasts are not efficient, even today! Thus, this always ends up with a quadratic unicast message complexity. This is one of the main downsides of PBFT.
- At most $f$ lying nodes can exist any point.
- Messages can be efficiently signed and parsed.
	- We need this in Byzantine scenarios, since a lying node can impersonate an honest node. We need signatures to make sure that this cannot be done easily.

We define the following message format:
$$
\langle m_1, m_2, ... \rangle_{\sigma_i}
$$
As a series of plain-text messages $m_i$, bundled into a single message (indicated by the brackets) and signed by a node $i$ (this is what $\sigma_i$ indicates).

Usually, we denote messages like $\langle \text{TYPE}, \text{field}_1, \text{field}_2, ... \rangle_{\sigma_i}$ which indicate a message of a particular type, and with arbitrary fields of varying numbers.

PBFT also defines **views**. Views, are essentially time intervals, denoted by an unsigned integer that the algorithm uses to synchronize messages. They are in nature, very similar to how *terms* were defined in Raft.
Each view has a predefined leader node as a primary. In a system with $n$ nodes, where each node is numbered from $1$ to $n$, the primary for view $v$ would be the unique node $i$ such that $i = v\mod n$. Thus, a view change translates one-to-one with a change of a primary node.

With the above, PBFT operates in 3 phases:
- `PRE-PREPARE`: Notify the system about new client requests. Broadcast messages to the whole system.
	- A `PRE-PERPARE` message is of the form $\langle\langle {\tiny{PRE-PREPARE}}, v, n, D(m) \rangle_{\sigma_p} \; , m \rangle$, where:
		- $v$ is the view number
		- $n$ is a sequence number, with a low and high watermark
		- $D(m)$ means the digest of $m$. 
		- $m$ is the actual message received from the client, which describes a record to push to our state machine.
	- In the above, $p$ is the primary node of the current view (which should be $v \mod n$, where $n=3f+1$).
- `PREPARE`: For replicas that accept the previous `PRE-PREPARE` message, this would be screamed throughout the whole system.
	- A `PREPARE` message from node $i$ looks like $\langle {\tiny{PREPARE}}, v, n, D(m), i \rangle_{\sigma_i}$
	- By looking at the fields $v, n$ and $D(m)$, we can decide which `PREPARE` message belongs to which `PRE-PREPARE` message.
- `COMMIT`: Any node that has accepted a `PRE-PREPARE` message, waits until it gets $2f+1$ `PREPARE` messages that match the `PRE-PREPARE`, and at that point, record is committed. A commit message is screamed throughout the cluster.
	- A commit message looks exactly like the `PREPARE` message, just with a different type field.

Thus, a normal operation of this system, where no view changes happen, would look like:
- A client reaching out to the primary node with a message $m$.
- The primary (which could be faulty!) is *supposed* to scream `PRE-PREPARE` messages in the cluster (of course, it may do other things if it is Byzantine).
- Each replica gets the `PRE-PREPARE`. A replica may reject the message (if it is Byzantine, or if it suspects that leader itself is Byzantine). A non-faulty node that rejects this message would **do nothing**.
- A replica that accepts a `PRE-PREPARE`, would scream `PREPARE` at the whole cluster, and wait for an extra `2f` messages of this kind to be reflected back to it. Thus by the end, it would have at least $2f+1$ `PREPARE` messages, with the aforementioned `PRE-PREPARE` message. **All of these are committed to the log** and only then, the message is committed.
- After the message is written to the log, we will scream `COMMIT` messages at the rest of the cluster.

"READ VIEW CHANGE FROM THE PAPER, I LITERARILY DID NOT UNDERSTAND THE PROFESSOR'S DESCRIPTION!"
