This note deals with an intro to Byzantine Fault Tolerance, and the main literature that shaped the understanding around the algorithms that attempt to handle this type of operating condition.

This note references multiple papers. This is done to make sure that just by reading this note, a basic understanding of the type of problems that need to be addressed is provided.

# Agreement And Faults

**Paper:** [Reaching Agreement in Presence of Faults](https://doi.org/10.1145/322186.322188)

The problem under consideration here, is the one that lies at the foundation of basically any consensus protocol (Byzantine or not). The problem is described as follows:

> A set of $n$ processes are configured in a network. For now, assume that the network is completely reliable and with negligible delay.
> Each process $p$, owns a private value $\mu_p$, which is not known to any other process, unless the process explicitly makes it known by sending a message to another process.
> Some processes (a set of at most $m$ processes) may be faulty, meaning that they might lie about the actual value of their $\mu_p$, or may send arbitrary messages to other processes. 
> We want to construct an algorithm that exchanges messages through a series of rounds within the network, and by the end of its execution (assuming there is one that is), each process can create a vector of values from other processes for which **Interactive Consistency (IC)** holds.

>[!IMPORTANT] Interactive Consistency (IC)
>Assume that a set of $n$ vectors $V_p$ is given. Each vector contains $n$ elements, where each element $V_p(p')$ corresponds with the value of a process $p'$. 
>Assume that $F$ is the set of faulty processes; IC holds if and only if:
>- $\forall p, q \notin F: V_p = V_q$ 
>- $\forall p, q \notin F: V_p(q) = \mu_{q}$
>
>Informally, IC holds if and only if all nonfaulty processes compute the same vector, and the entries of that vector that correspond to nonfaulty processes, equal to the private value of each process.

## Single Fault Case

Let's start with the case that $n=4, m=1$, meaning that a network of 4 nodes exist where at most one of them is faulty.
Again, faulty means that node can (among other things ...):

- Lie about its value, or the values that it has heard about other processes
- Refuse to answer
- Send gibberish to other processes

So, with this in mind, here is an algorithm:

- Operate on 2 rounds of exchanges:
	- Scream your private value at all other processes. If a process does not answer you, generate a random value and assume that was sent from it instead.
	- Construct the initial $V_p$ value from the results that you hear from others.
	- Scream the $V_p$ value at others
	- Record the values that you receive from others. If some process does not respond, create a random vector for it and use that.
- By the end of the above, you will have received 3 vector values from the other 3 processes. Construct the final vector value $V^*$ by setting:
	- If you are process $p$, set $V^*(p) = v_p$, i.e. the entry corresponding to yourself will be your private value by default.
	- For any other process $q$, and any process $t \neq p$, examine the values $V_q(t)$, and if at least 2 of them are the same, pick that value.
	  If none of them agree, use a predefined, placeholder value (like `NIL`)

So, why is this correct?
If process $q$ is nonfaulty, any other process $p$ will see the value $\mu_q$ at least 3 times for it:
- One in round 1, when $q$ straight up tells $p$ what its private value is (although we won't actually use this)
- At least 2 times in round 2:
	- 2 times from the other two, nonfaulty processes
	- Perhaps another time from the faulty process if it plays nice
Thus, majority of the values for $q$ will agree on $\mu_q$ and we are OK.

However, if process $q$ is faulty, we should make sure that any other nonfaulty process $p$ agrees on the same values for $q$.
If the value agreed upon by all nonfaulty processes is `NIL`, then we are done. If not, assume that some value $v$ has been agreed upon in a nonfaulty process $p$. Note that since $p$ is nonfaulty, then it must be the case that this value was received from at least 2 processes in round 2.

Since $q$ is faulty, there is not much sense betting on a value that it sent in round one (which is why the value that we record for the vector in round 1 is discarded). If $p$ is nonfaulty, then it must have received the value $v$ from at least 2 processes like $p'$ that $p' \neq p,q$. This can only happen if these two processes, received the same value from $q$ in round 1, and as such each of them also must have received the same value at least 2 times (including from $p$ itself perhaps). Thus all nonfaulty processes will decide on the same value $v$. Whether that is the actual private value, or the same random value if we are very lucky is unknown.

## For Multiple Faults

Now, assume we have general $n,m$, with the added condition that $n \geq 3m+1$. We will now define an algorithm that solves the problem in $m+1$ exchanges.
We need a new definition. 

Assume the set of processes is $P$ with $|P| = n$, and the set of all possible values that a process can propose is denoted by $V$.

Define $W^{(P)}_k$ to be the set of nonempty lists of length at most $k+1$, with elements picked from $P$. Then we define a $k$-level scenario $\sigma$ as a mapping from $W_k$ to $V$.

Now, by definition of $W^{(P)}_k$, we see that:
$$
\forall w \in W^{(P)}_k: \exists\; 1 \leq r \leq k+1,\; p_i \in P: w = p_1p_2 ... p_r
$$
For each $k$ level scenario $\sigma: W^{(P)}_k \rightarrow V$, we set $\sigma(w)$ as the value that $p_2$ tells $p_1$, that $p_3$ told $p_2$, that ..., that $p_{r}$ told $p_{r-1}$ is $p_r$'s private value $\mu_{p_r}$.
Essentially, each value $\sigma(w)$ represents a chain of message bounces between processes.

With the definition of $\sigma$ above, if $q$ is a truthful process and $p$ and $w$ are arbitrary, then we have the invariant that:
$$
\sigma(pqw) = \sigma(qw)
$$
Which essentially means that once a nonfaulty process hears of some value, it correctly tells it to another process. From the definition above, it is clear that the set of messages that a processor $p$ receives in a set scenario $\sigma$, would be the range of $\sigma_p$. the restriction of $\sigma$ to string that begin with $p$. Note that be definition, $\sigma_p$ can be calculated at $p$ by just looking at what we get during this scenario. Thus, it is with this $\sigma_p$ that we will proceed to define how to decide about the vectors.

The procedure for recording the value of a process $q$ in the vector constructed at process $p$ is as follows:
- If there exists $Q \subseteq P$ such that $|Q| > (n+m)/2$ and some $v \in V$ such that:
  $$
	\forall w\in W^{(Q)}(m-1): \sigma(pwq) = v
	$$
  Then record $v$ for $q$.
- 