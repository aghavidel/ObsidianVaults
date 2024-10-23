# DCTCP (Cont.)

We were discussing how `ECN`s were treated last time.

## Using `ECN`s

When the receiver sends marked ACKs with the `ECN` flag set, the source is able to deduce exactly how many packets saw congestion. We also discussed the state machine that lets you do this with delayed ACKs.

Now, the main idea is to estimate the probability of congestion on the sender side, so we can act accordingly. To do this, we introduce two parameters:
- The sender maintains a congestion probability of $\alpha$
- The switch, maintains a minimum congestion queue length of $K$. We assume this is constant. If a packet arrives and sees a queue larger or equal in length to $K$, it is marked with `ECN`

On the sender side, $\alpha$ is updated once for every window of data (roughly each RTT) as:
$$

\alpha \gets (1 - g)\times \alpha +  g \times F
$$
Here, $F$ is the ratio of bytes with `ECN` set to the total number of ACKed bytes. The gain parameter $g$ has the same semantics that we had for the same gains used for setting the smooth RTT values.

As the connection progresses, the value of $\alpha$ approximates the probability that the queue on the bottleneck link is larger than $K$.

The other difference in the sender is how we react to changes in $\alpha$. TCP is very pessimistic, and upon seeing the `ECN` bit, it will chop the window in half. In DCN, this is just too wasteful, so DCTCP is a bit more optimistic and slowly lowers the window as $\alpha$ increases:
$$
\begin{align}
	\text{cwnd} \gets & \text{cwnd} \times (1/2) & \text{TCP} \\
	\text{cwnd} \gets & \text{cwnd} \times (1 - \alpha/2) & \text{DCTP}
\end{align}
$$
So DCTCP is still an AIMD protocol, but the decrease is much more gentle than just chopping the window in half immediately.

## Analysis

The analysis of DCTCP is interesting, so lets look at it.
The main setting that we are worried about is Incast, which happens when $N$ senders are synchronized over the same bottleneck link with limited capacity $C$. This means that all senders exhibit the same window function $W(t)$ and thus the queue length $Q(t)$ would be:
$$
Q(t) = N.W(t) - C.RTT
$$

![[Pasted image 20240324190610.png|500]]

Essentially, this means that the queue length exhibits the same sawtooth behavior that the TCP congestion window does. The right figure shows us the parameters that we must control:
- $Q_{max}$, the maximum queue length
- $T_c$, the duration of oscillations
- $A$, the amplitude of oscillations, which is our main worry

Now, while we are on the linear region of the $W(t)$ function, we define $S(W_1, W_2)$ to be the number of packets transmitted when we start with a window size of $W_1$ and end with a window size of $W_2 > W_1$. Since the growth is linear, it is easy to show that:
$$
S(W_1, W_2) = \frac{W_2^2 - W_1^2}{2}
$$
Now, we are concerned about when the switch starts marking the packets. That happens when $Q(t_0) = K$, so we have:
$$
W(t_0) = W^* = \frac{C\times RTT + K}{N} \equiv \frac{BDP + K}{N}
$$
Now, for the first RTT after $t_0$, we have $S(W^*, W^*+1) = W^* + \frac{1}{2}$ marked packets. Now, assuming this is the steady state, the last time this happened, it caused the window to get updated such that:
$$
\begin{align}
W_{start} = (W^* + 1)(1 - \alpha/2) \Rightarrow \text{Total Packets} = & S(W_{start}, W^*+1)\\
= & S((W^* + 1)(1 - \alpha/2), W^*)\\
\end{align}
$$
Which gives:
$$
\begin{align}
\alpha = \frac{W^* + 1/2}{\text{Total Packets}} & = \frac{W^* + 1/2}{S((W^* + 1)(1 - \alpha/2), W^*)}\\
& = \frac{W^* + 1/2}{(2W^*+1-\alpha/2\times W^*).(-1 + \alpha/2\times W^*)/2}\\
& = \frac{2W^* + 1}{(\alpha W^*/2 - 1)(2W^* + 1 - \alpha W^*/2)} \\
\end{align}
$$
Letting $\beta := \frac{W^*/2}{2 W^* + 1}$, this can be rewritten as:
$$
\alpha^{-1} = (1 - \alpha \beta)(\alpha \beta - \frac{1}{2W^* + 1})
$$
For large values of $W^*$, which can happen in a DCN, we would have $\beta \approx 1/4$ and so this simplifies to:
$$
\alpha^2(1 - \alpha/4) = 2/W^*
$$
A first order approximation of this would be to just have $\alpha \approx \sqrt{2/W^*}$, which is not a bad approximation according to NS-2 simulations.

It is easy to find the values of all other desired variables now:
$$
\begin{align}
A & \approx N/\alpha = N\sqrt{W^*/2} = N\sqrt{\frac{BDP + K}{2N}} \\
T_c &= RTT \times (\alpha W^*/2) \approx RTT \times \frac{A}{N} \\
Q_{max} &=N\times(W^*+1) - BDP = K + N
\end{align}
$$
This is fantastic! It means that with DCTCP:
- Tuning $K$, tunes the maximum queue length if we have an upper bound on the number of concurrent connections.
- Oscillations are stopped much faster than TCP. In TCP, oscillations are stopped in $O(BDP)$, while here, they last $O(\sqrt{BDP})$.

The biggest thing with DCTCP is that it maintains very small queues, and thus, **the latency of flows rises very slowly with more connections on the same link!!**

## Parameter Estimation

To keep the above analysis valid, the parameters $K$ and $g$ should satisfy certain constrains. Specifically, we should have:
$$
\begin{align}
Q_{min} = Q_{max} - A \gt 0 & \Rightarrow K \gt BDP/7 \\
\end{align}
$$
Also, if the gain $g$ is too large, we can perform even worse than TCP itself! 
The worst case scenario is that $K$ is so small that we are getting congestion throughout all of the oscillation period $T_c$, which corresponds to when $W^* = BDP+K$ and $N=1$ and $Q(t) = K$ (i.e. even the smallest increase in the window will cause a packet to be marked). We want to make sure we are not as aggressive as TCP, chopping the window in half, so:
$$
(1 - g)^{T_c} > 1/2 \Rightarrow g \lt \frac{1.386}{\sqrt{2(BDP + K)}}
$$
So essentially, to compete with the same bandwidth of TCP on a link:
- DCTCP needs larger $K$ for larger BDPs, and that is understandable 
- It needs *less* gain for larger BDPs, it needs to be much more gentle with lowering the window size

# Swift

DCTCP was designed at Microsoft, but around the same time, Google was also thinking about what sort of transport to use in a DCN. While DCTCP works by controlling queue length on the switches, Swift is part of the good old delay-based CCAs that we have seen, but unlike say, TCP, where we wait for packet drops to react, Swift reacts to **increases in RTTs**.

This design, instantly relegates it to the DCNs, since such a protocol design is not very good for WANs, where RTTs are particularly wild functions, but in DCNs, as long as the RTT measurement is done with good enough accuracy, then such a protocol will work well.

## Design Goals

Swift pursues the same goals that DCTCP does, low tail latency for short lived flows in a DCN and high throughput for background flows. However, Swift assumes no support from the network fabric, so it does not use ECNs, and instead, it only relies on the host networking stack (i.e. it only assumes hardware support from the host NICs).

The main support that Swift needs, is NIC level timestamping, where NICs can timestamp packets on ingress/egress. This is key in making sure that RTT measurements are accurate. 
This cannot be done in normal host stacks, since it comes at a significant CPU cost if it cannot be offloaded to the NIC.

![[Pasted image 20240324211903.png|500]]

Swift does not use the Socket API that TCP relies on, it has its own custom API for programming transport, and is very similar to how RDMA works (although, that is where the similarities end, Swift does NOT use RDMA).

But basically, applications register *operations* (such as `send`s or `recv`s) on operation queue which is managed by an operation *scheduler* in the host network stack. The kernel layer of the stack will keep track of what flows should go to which operation, and the Swift implementation will configure the transport based on the received packets for each flow.

Now, delay based CCA is nothing new, TCP Vegas in particular is quite similar to Swift in this regard, but what makes Swift work, is two things:
- Swift recognizes that end-to-end delay is NOT just because of networks, there is usually a delay from the host in it as well. Consider the following scenarios:
	- A host is under heavy load, and upon receiving a packet, it takes a long time to ACK it
	- A network is under heavy congestion, and upon receiving the packet, the instantly emitted ACK is delayed
- Most CCAs cannot distinguish between these two scenarios, and thus, for the first scenario, the will drop the window significantly which won't actually help that much.
- **Swift uses NIC level timestamping to distinguish between host and network delays**
