**Paper:** [Understanding TCP Incast Throughput Collapse in Datacenter Networks](https://dl.acm.org/doi/pdf/10.1145/1592681.1592693)

This is very much related to the DCTCP lecture in CS 659.  The problem of Incast though is quite general, and worth discussion.

Incast, refers to a pathological congestion control behavior, where a receiver waiting for responses from several senders, will observe a throughput collapse on a shared link. This means that on a 1 Gbps link, the goodput is much lower, despite high utilization of CPU resources on the senders.

This means that some packet drops or loss is happening on the link, and that is true. The paper makes a thorough discussion of Incast with regards to TCP, and it is quite well written, so we will discuss it at length.

# Initial Tests With TCP

The main thing to consider is application-level throughput (goodput), defined as the total number of bytes received by all senders, divided by the finishing time of the *last* sender (same that we do in NS-3!).

Their testbed consists of a variable number of senders, creating TCP connections to the same application on a remote receiver. The number of bytes that each connection must send have been fixed. The system is evaluated by starting the connections at the same time and timing everything until all senders finish.

The initial results look like the following:

![[Pasted image 20240324160524.png|500]]

Looking at TCP sequence numbers also gives some interesting insights:

![[Pasted image 20240324160656.png|500]]

This shows that (at least for these two senders), they experienced a TCP timeout at the exact same time. This TCP timeout take about 200 milli-seconds to happen (this is the default RTO value on Linux). The first thing to note however, is that this timeout value is just way too large.

In a DC, RTTs are typically less than a milli-second, these days, they take micro-seconds. Even in this configuration, there is barely any packet that has an inter-arrival time more than 200 milli-seconds;

![[Pasted image 20240324163455.png|500]]

As you can see, the buckets above 100 are all nearly empty.
The authors experimented with several different ways of just fiddling with TCP parameters to see what happens, and the only modification that was *somewhat* helpful, was reducing the maximum RTO value from 200 milli-seconds, down to some other reasonable value.

Playing with different RTO values, reveals a curious behavior in the goodput plot:

![[Pasted image 20240324163756.png|500]]

As you can see above, there are three regions, marked $R_1$, $R_2$ and $R_3$.

- In $R_1$, there is a sharp decline of goodput until it reaches a minimum value. Where this minimum occurs is quite consistent, always around 7 or 8 senders (keep that in mind!).
- In $R_2$, there is a recovery period, where the goodput increases. The increase is faster with lower minimum RTO values.
- In $R_3$, there is again a slow decline of goodput, and things only get worse afterward. Where this region starts and the previous one ends, is not entirely consistent.

So, how to make sense of this?
# Model

To model this, we can just start with the goodput definition. In all of these tests, each sender must transfer 100 blocks of 256 KB each, a total of 25 MB of data. The total amount of data to be transferred, $D$, divided by the time it takes to do it, $L$, gives the goodput as $\frac{D}{L}$.

Now, when we add timeouts, we get a bit of a different formula. If $R$ timeouts happen, each taking $t_i$ seconds for $i \in \{1, 2, ..., R\}$, then we would have:
$$
g = \frac{D}{L + \sum_{i=1}^{i=R} t_i}
$$
Now, most timeouts are implemented with exponential backoff, which means that $t_i = r . 2^m$ where $m$ is the number of timeouts that immediately preceded this timeout and $r$ is some initial value, usually set to the maximum timeout value (i.e. the 200 milli-second value that we saw previously). 

As a first approximation, and not a bad one in DCN context, we assume back-to-back timeouts do not happen. So we would have the following for $S$ senders:
$$
g = \frac{S.D}{L + R.r}
$$
Now, we are stuck. The values of $D$ and $r$ can be fixed, but $L$ and $R$, both depend on the number of senders, $S$, and the link properties. 
The authors empirically fit a curve for the number of timeouts, $R$, like the following:

![[Pasted image 20240324170558.png|500]]

We let:
$$
R = 
\begin{cases}
3.5 \times S & S \leq 10 \\
35 & S \gt 10
\end{cases}
$$
Which smacks of giving up a bit too soon, but we let it be for now. 
Now for $L$. Since the link is shared, there will be contention, and thus, there will be a measurable inter-arrival time between packets $p_i$ and $p_{i+1}$. Assuming that timeouts are rare (which is true in general, especially in a DCN context), TCP will operate near full window, where each packet would be around the maximum segment size allowed on that system $M$. The authors used a value of 1448 for this.

Thus, on a link with capacity $C$, the expected time it takes for things to finish up for a sender would be:
$$
\mathbb{E}[L]  = \frac{D}{C} + \frac{D}{M}.\mathbb{E}[I]
$$
Where $I$ is the interarrival time of packets. Most flows will obey something resembling a Poisson process, and thus, $I$ would be close to an exponential variable. The mean of this variable depends on the number of senders, and thus, the authors again turn to curve fitting:

![[Pasted image 20240324171708.png|500]]

They let:
$$
\mathbb{E}[I] = \begin{cases}
0.45 \times S & S \leq 10 \\
4.5 & S \gt 10
\end{cases}
$$
And thus gives a curve pretty close to what we saw.
The big picture though, is the following:
1. With multiple senders on a shallow buffer switch, the buffer is quickly filled and multiple timeouts happen, which forces TCP to throttle the stream. Since this happens synchronously for all connections, the goodput collapses.
   Having more fine-grained timeouts would help, since it can break the synchronization between multiple senders. 
2. As the number of senders increases, it becomes less probable that all of them timeout near the same time, so the goodput would again improve a bit.
3. At a certain point, the link becomes so congested that inter-packet wait time becomes comparable to the RTO value, and at this point, having more senders would collapse the goodput slowly.
