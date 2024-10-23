# Intro.

We describe multiple TCP congestion control protocols here, we assume that we already remember everything from [[TCP]] note.

## Some Definitions

Most of these were discussed in our [[TCP]] note, but that aside, let's do a little recap:

### Segment Definitions
- **SEGMENT**: Any TCP/IP data or ACK packet from the upper layer.
- **SENDER MAXIMUM SEGMENT SIZE**: Also `SMSS`, is the maximum size of a segment that a TCP sender can send (not including headers).
- **RECEIVER MAXIMUM SEGMENT SIZE**: Also `RMSS`, is the maximum size of a segment that a TCP receiver can accept. Either negotiated during connection startup or given a default value of 536 bytes.
  Similar to above, it does not include header size.

### Window Definitions
- **RECEIVER WINDOW (`RWND`)**: The most recently advertised receiver window.
- **CONGESTION WINDOW (`CWND`)**: A variable that limits the amount of data that TCP can send.
- **INITIAL WINDOW (`IW`)**: The value of `CWND` directly after 3WHS.
- **LOSS WINDOW (`LW`)**: Size of `CWND` after a TCP detects loss using the retransmission.
- **RESTART WINDOW (`RW`)**: Size of `CWND` after TCP restarts transmission after an idle period.

### Other Definitions:
- **FLIGHT SIZE**: The amount of data sent, but not yet acknowledged.

## What A CCA Needs

Jacobson and Karels give a very thorough exploration on *what* a good congestion control protocol needs, and while most of it may seem obvious, it's not bad to reconsider them here as well, if only to review a landmark paper about congestion control.

The full paper is [here](https://ee.lbl.gov/papers/congavoid.pdf), here we'll provide a summary.

The authors first explain that for the network to experience a robust equilibrium of data exchange, it should ideally behave like how liquid flows in a network of pipes, it should obey a conservation law that states that for a system in equilibrium, a packet is not emitted until an older packet has exited the network.

A quick glance at the internet, with it's many overloaded routers will tell you that this isn't the case. The authors recognize that by the fluid analogy, there can be only 3 causes for this:

- The connections never manage to reach equilibrium.
- Some transmitters emit new packets while older ones are still in the network.
- Some resource limit on the path prevents any equilibrium from happening.


So one way to tackle the problem of congestion control is to devise an algorithm for each of these problems, and then, glue them together into some *base* that our CCAs will use.

### The First Problem

For a connection to be able to reach equilibrium, we need to make sure that it does not exceed that available link bandwidth, and if possible minimize packet loss.

Now packet loss can have stochastic behaviors that are completely out of our control, so we won't be getting rid of it any time soon. So for the most part, the main problem here boils down to making sure that we act appropriately when a loss occurs, and to also keep the sender on a short leash when the transfer is just beginning.

One way is to *probe* for bandwidth, which means to start with small enough values and then increase the rate until we approach that threshold where we are utilizing enough bandwidth, but are also trying to minimize packet loss. To this end, ACKs are used to create some sort of self-clocking system.

![[Pasted image 20221001013555.png|600]]

The figure above shows why this would work. Making the simplifying assumption that the paths taken by data packets and ACKs have similar properties, in both paths, the available bandwidth is dictated by the slowest segment of the link.

While the sender can send an entire train of packets successively, they don't arrive the same way to the sender, they are sure to experience some amount of empty space between each other due to the bottleneck in the middle, and so the same thing goes for ACKs.

This also gives rise to a measure of how *open* the links are called **Bandwidth-Delay Product** or BDP, which is essentially just $R.D$ where $R$ is the rate at which we can send bits in the channel (i.e. the channel capacity) and $D$ is the Round-Trip Time (RTT) of the link, the time from when a packet is emitted and it's ACK is received.

The BDP gives the maximum amount of inflight bits at that time, the TCP cannot exceed this value at any time.

Since ACKs are pretty small, not much really happens to the ACKs, and they arrive with the same amount of delay between each other, which we have denoted with $A_b = A_s$, and so the rate of received ACKs is directly proportional to the available bandwidth, and so we can get a sense of it by just putting a train of data into the network and observing the rate at which we receive ACKs.

One solution that takes advantage of this fact to probe for available bandwidth (which might seem trivial, and it kind of is), is what we call the *Slow Start* algorithm.


#### Slow Start Algorithm

We maintain a congestion window, or `CWND` variable in the states of each connection endpoint (so one for the sender and one for the receiver). Now:

- After a reset (`RST`) or a loss, set the window to 1 packet.
- On each ACK, increase `CWND` by 1 packet.
- Take the minimum of the windows that the sender and the receiver have advertised, and this will be maximum number of packets that can remain without ACK at that point.

While it is called "slow" start, it really isn't slow … look at this:

![[Pasted image 20221001014905.png|500]]

Assuming no loss occurs, if we look at the sender between RTTs, we'll see that for the first one, it sends only one packet and then refrains from sending another. When it receives the ACK of the first packet (which are shown with small white boxes carrying the sequence number) we'll see that it will now send 2, since the ACK of SQN 1 has been received, so while we send SQN 2 like usual, we also move on to SQN 3, and so we have 2 inflight packets.

During the next RTT, we receive ACKs for the SQNs 2 and 3, moving on to 4 packets … and so on. While it is called Slow start, it really isn't that slow in the first place. As you can see, the window is increasing *exponentially*. This means that the protocol will have somewhat of a negligible effect on performance, but it also means that it is technically possible for the algorithm to overshoot the maximum possible rate over the link by two folds!

On the other hand, you can notice the some packets are stacked on top of each other. This is here to convey that the packets are essentially sent in such a way that requires at least one of them to be queued when it reaches the bottleneck.

>[!FAQ]- Why?
>Well the reason is that each ACK essentially corresponds with the sender emitting 2 packets back to back (one in response to the ACK to keep the channel utilized, another because the ACK let's loose another packet by increasing the window).
>So we are effectively sending more than the expected rate of bottleneck, so at least one of these will be queued.
>
>This is concerning, since while this means that the algorithm has negligible costs in performance, it has exponential short time utilization of bottleneck queues! If we were to emit packets in a window of size $W$, we'll end up with queues of lengths $\frac{W}{2}$ at the bottlenecks.

#### What If We Didn't Use Slow Start?

This happens!

![[Pasted image 20221001163157.png|550]]

>[!NOTE]- How To Read This Graph
>Each point is 512 Bytes of data. The x axis shows when the packet was emitted, and the y axis shows the sequence number of packets inflight. Packets sharing the same SQN, but different times, have been retransmitted.
>
>This model attempts to send a whole windows worth of packets as soon as possible, effectively skipping the *slow* part of Slow-Start. 

In an ideal situation, we expect something along the lines of the dotted line, a line with a fixed slope, equal to the bandwidth of the bottleneck, with minimum deviation, but what we are seeing is nothing like that!

There are multiple retransmissions, and significant amount of available bandwidth is wasted on retransmissions, an amount that we can estimate by regressing a line through the data and calculating the slope, finding out how much it differs from the actual ideal slope. Doing so yields a value of almost 40 percent of the ideal value, more than half of the bandwidth is wasted!!

If the differences between the input rate and the link bottleneck is *too high*, there is the possibility of a deadlock, where the connection get's trapped in this loop of sending and resetting.

The slow-start algorithm provides some solution to this problem, by sacrificing some initial available bandwidth, the sender approaches to the ideal bandwidth that the link can provide, until it reaches a certain point when `CWND` is much larger than the receiving window (`RWND`) and so the receiver dictates the rate of data … and that's what we want (though there is a few caveats to that, but we'll get to it).

So the following is the figure above, but corrected (at least in the beginning):

![[Pasted image 20221002000152.png|550]]


### The Second Problem

Assuming that we solve the previous problem, this problem (i.e. sending new packets while old ones are inflight) means that for whatever reason, the sender's retransmit timer is misbehaving. If the timer goes off too soon, the sender can resend packets erroneously, something that can flat out cause the connection to fail instantly.

To solve this, a good *RTT Estimator* is needed, something that can keep track of the round trip time fairly well. One thing to note here is that since RTTs can behave stochastically, we can't just live with a single population measure among RTTs, the *variation* of RTTs is also important.

This is especially important when a network is under high load, where queuing theory tells us that the service time (essentially the RTT) and the variation of it, scale with $\frac{1}{1 - \rho}$ where $\rho$ is the system load (the ratio of arrival and departure rates).

One of the most common approaches to calculating the RTT ($R$) from measurements ($M$) is a simple low-pass filter:
$$
R \leftarrow (1 - \alpha) R + \alpha M
$$
The value of the retransmission timer (`RTO`) is some function of this estimation (the value of $\alpha$ is suggested to be pretty low, giving the estimation the name Smooth RTT or simply `SRTT`, around 0.1, and the `RTO` is set to be around double the amount depending on the value of the variance).

In the early versions of TCP, the `RTO` was simply set to be $\beta R$ for some $\beta > 1$, usually 2. But that is not a good way to implement this, a fixed value is too forgiving in low load and quickly becomes problematic when the load exceeds a certain value, due to the $\frac{1}{1 - \rho}$ behavior that we discussed.

The value of $\beta$ needs to adapt based on the variations of $R$, so one obvious candidate to add to the equation is the running variance of $M$, but the problem is that it involves computing a square root, which can be slow on dedicated router hardware per packet. We really only need a close estimate of it. 

Jacobson and Karels instead suggest we use the *mean deviation*:
$$
mdev[M] = \mathbb{E}[|M - \mathbb{E}[M]|]
$$
This has two advantages:
- It computes easily, since it can be updated per packet using the same low pass filter that we used for $R$.
- It actually overshoots the value of the variance, since we have:
$$
mdev[M]^2 = (\mathbb{E}[|M - \mathbb{E}[M]|])^2 \geq \mathbb{E}[|M - \mathbb{E}[M]|]^2 = \sigma^2
$$
So in practice, using a separate gain of $\beta$ for this estimation, we'll get the formulas in [[TCP#Subsequent Measurements]].

The value of 4, in $SRTT + 4 . RTTVAR$ for the `RTO` value, has some rationale behind it.

>[!FAQ]- Rationale Of The "4" Used Above
>The dominant mechanism that effects RTTs is mostly two folds in networks. They are either *delay bound* or *bandwidth bound*. 
>Network paths bounded by delay have RTTs that are determined by the store-and-forward actions of their routers, NOT the bandwidth. On the other hand, some networks are completely bound by bandwidth (especially during heavy loads).
>Our concern is the latter, since in a bandwidth bound path, the average packet size that traverses the bottleneck is the main thing that determines the RTT.
>If we increase the window by $\Delta w$ on such a path with bandwidth $B$, what we expect is:
>$$
>\Delta R \approx \frac{\Delta w}{B}
>$$
>Such a change may not be immediately reflected in the RTT variance, we can end up with erroneous timeouts here!
>In the worst case where the delay is entirely the fault of the window, since the window doubles, the RTT also approximately doubles. This means that during round $i$, we'll have that $R_i = 2R_{i-1}$, and so $\Delta R_i = R_i - R_{i-1} = R_i/2$.
>Taking this into account, using a simple coefficient of $c$ for this, we end up with $RTO_i = R_i + c\Delta R_i$, and so we have $RTO_i = R_i (c/2 + 1)$.
>To make sure we don't have spurious timeouts, we need to have $RTO_i > R_{i+1}$ and if the path model continues to hold, we'll have $RTO_i > R_{i+1} = 2R_i$ and so we have $c/2 + 1 > 2$ and thus $c > 2$.
>The value $c=4$ has the benefit of not needing a multiplier at all, since it can be multiplied by just 2 shifts.

To compare these two approaches, here is a comparison:

This is the `RTO` plot without using the variance (i.e. fixed $\beta$):

![[Pasted image 20221002013731.png|600]]

This is the `RTO` plot wile using the variance:

![[Pasted image 20221002013813.png|600]]

The thin lines are the real RTT values, and the thick lines are the timeout estimates.

As you can see, the `RTO` is following the RTT closely, and it also barely overestimates it most of the time and rarely underestimates it.

#### The Need For Backoff

There is still one question about this. How many times, and with what spacing should we attempt to resend packets that timeout?

>[!IMPORTANT] Backoff Timers
>For an arbitrary network, of unknown topology and unknown number of competing conversations in a network, **only** exponential backoff can keep the network stable while simultaneously attempting to resend the packet. 
>Anything faster can destabilize the network, assuming that packet loss is due to congestion and not some checksum error. Theoretically, for a network of infinite nodes, even exponential backoff isn't safe.

Proof of this is much beyond this little note, you can see [this](https://www.jstor.org/stable/2345773) for the proof, but in essence, it has the same rationale of the exponential backoff in [[ALOHA]].

### Third Problem

Assuming we solved the previous problems, at this point we can safely say that if the timer has expired, we have lost a packet, and not been fooled by a broken timer.

Packets get lost for 3 reasons:
- They end up in the wrong place, which isn't our problem, it's a problem with routing, so we can simply ignore it.
- They get damaged.
- They are dropped from queues due to congestion.

The second problem happens very rarely, and there isn't much that a protocol can do to solve it, other than using ECCs which are just too much of an overhead for per-packet hardware. So we'll have the 3rd case on our hands. It can be shown that as far as the scheme below is concerned, very high error rates are required to actually cause the connection to collapse.

Whether or not the congestion is because of *this* connection or another is hard to tell, the only thing we know is that *someone* is, and so as long some measure of avoidance is implemented for each participant, we can hope that we can alleviate the congestion.

#### Congestion Avoidance Algorithm

We know how to signal congestion somewhat reliably, by looking for timeouts, but what to do exactly?

We won't go into details, we'll just give intuitions:
- When a network is congested, a simple model for the load of a network when sampled at the beginning of RTT $i$, is $L_i = N + \gamma L_{i-1}$, where $L$ is the load of the network and $N$ is some constant that dictates the steady state load of the network and is assumed to change only within the longer periods of time. The coefficient $\gamma$ is a variable in time that dictates how much we are getting close to congestion. 
  It is not hard to see that if $\gamma < 1$, then we won't reach a *collapse* (i.e. a point where $L$ grows without bound), but if $\gamma \geq 1$ (and there is no reason why it should not), we are in trouble, and $L_i$ grows exponentially, since at some point $N$ is dwarfed so much by the rising value of $L_i$ that we essentially just have $L_i \geq \gamma L_{i-1}$ and we end up with $L_i \geq L_0 \gamma^i$.
  Once we reach this point, the queues also start filling up exponentially. It is not hard to see that when this happens, the only case that let's us stop this collapse is to throttle back the traffic *at least* as quick as it grows, so we need an exponential backoff of traffic window:
  $$
  W_i = d\;W_{i-1} \quad \quad (d < 1)
 $$
  Where $W_i$ is the sender's congestion window when attempting the $i^{th}$ packet transmit (so this means that the instant that a loss is detected, this formula takes effect). 
  There is reason to believe that when the internet grows large (it may already have), we'll need to consider a second order model for the load on the network unlike the first order model we used. To see how that fairs against what we discussed, you can just read any good Linear Control Theory book (spoilers, it ends disastrously).
- When a network is far from congestion, the connections need to probe for available network bandwidth on their side to keep the link's utilized. This needs to be done very carefully though. We'll not give any justification here, but the best policy for increasing the window here is linear one:
  $$
  W_i = W_{i-1} + u \quad \quad (u \ll W_{max})
 $$
  Here, $W_{max}$ is the maximum value of the window for the current measurements (i.e. it's the same is the current BDP, minus the header lengths). 
  There is still a little catch, the increment of the window size should be done by an amount of $u$ *after each RTT*, but ACKs are received in lesser time intervals. In order to make sure that this formula is clocked nicely with the RTTs, we increase the window size by $\frac{1}{CWND}$ instead.
  This means that the acknowledgement of only the whole window, can allow for an increase in the window size.

Now that we know what to do, let's see how it is implemented.

## Slow Start and Congestion Avoidance Algorithm Implementations

The state variables `CWND` and `RWND` are maintained. While both endpoints should have these variables in case of a bidirectional connection, for a single directional connection, `CWND` is maintained at the sender and `RWND` is maintained at the receiver.

As per their definitions, it is not hard to see that the minimum of these two values, governs the number of inflight segments, and thus the throughput of the connection.

Another state variable that must be maintained is the Slow Start Threshold or `SSTHRESH`, which we'll discuss below.

### Beginning Transmission (In Modern TCPs)

After `SYN/ACK` has been sent and received (remember that these segments *do not* contribute to any window updates!), the initial window (`IW`) should be set according to the following:

```
IF SMSS > 2190:
	IW = 2 * SMSS
ELSE IF (SMSS > 1095) AND (SMSS <= 2190):
	IW = 3 * SMSS
ELSE:
	IW = 4 * SMSS
```

>[!FAQ]- Why Do It Like This?
>The full rationale of this choice is described in [RFC 3390](https://www.rfc-editor.org/rfc/rfc3390) , we'll give a summary of it here.
>TCP implementations may use any initial window size as far as most networks are concerned, but an upper bound is usually put on the initial window to prevent the connection from immediately overwhelming other new connections.
>
>To this end, a maximum number for the initial window is usually imposed on most TCP implementations.
>
>Now, as you probably remember, IP headers with no options are 40 bytes, and the most widely accepted value for the maximum transmission unit of a packet is about 1500 bytes. So if no TCP options are used, the TCP segment that is converted into a  1500 byte IP datagram has to be about 1460 bytes in total. The document suggests that the initial value should be 3 segments to allow for 3WHS to complete as soon as possible, so 3 times 1460 gives 4380.
>
>To allow for piggybacked `SYN/ACK` segments, we also need to have at least 2 times this value, meaning 2920 bytes. It is not hard to see that one way of expressing this would be to let:
>$$
>\text{IW} := \min(4\times\text{SMSS}, \max(2\times\text{SMSS}, 4380))
>$$
>You can check that the IF-ELSE conditions above correspond to this expression.

Now, depending on the value of `SSTHRESH`, which at first can be arbitrarily high to make sure that it is the *network* that dictates when a loss happens instead of the hosts, if `CWND < SSTHRESH`, we'll use *Slow Start* and if `CWND >= SSTHRESH` we'll use *Congestion Avoidance*.

### Slow Start

During this phase, the TCP increments the window by `SMSS` for each ACK until it either exits this phase or a loss is detected, but for some security considerations (which we'll leave for now), we'll do this instead:
$$
\text{CWND}_t = \text{CWND}_{t-1} + \min(\text{SMSS}, N) 
$$
Where $N$ is defined as the number of bytes that the incoming ACK just acknowledged. This is done to prevent the receiver from mounting an *ACK DIVISION* attack, where the receiver sends multiple ACKs that only acknowledge a small portion of each segment, and thus it will inflate the congestion window with this strategy (there is more to it than that, but we won't discuss it).

Hence from now on, **all windows are measured in bytes instead of segments**.

### Congestion Avoidance

Similar to above, during congestion avoidance, for each ACK, we perform:
$$
CWND_t = CWND_{t-1} + SMSS \times (SMSS / CWND_{t-1})
$$
This essentially means that after getting the full ACK of a window, we are allowed to send one additional segment-worth of data.

>[!WARNING] Silly Window Problem
>Using systems that have smaller MTUs, this action might cause extra fragmentations, and really degrade the performance. This is called the *Silly Window* problem, and it really isn't something that TCP handles, it's the job of higher layers and the hardware specification to handle this.
>
>For example, if the window comes to 1600 and the MTU is 1500 bytes (ignore headers), normally this would cause the packets to be fragmented to 1500 and 100 byte segments, but two 800 fragment packets will yield better utilization, since it prevents other connections from "stealing" too much bandwidth when we send a small 100 byte packet.

Now that we have these fundamentals out of the way, let's talk about the individual TCP algorithm "flavors".


## Tahoe 

Probably the most simple of the full TCP specs, Tahoe is essentially Slow Start + Congestion Avoidance + Fast Retransmit.

Specification can be found in [RFC 2001](https://www.rfc-editor.org/rfc/rfc2001), but in brief:
- Initialize `CWND` to 1 segment and `SSTHRESH` to 65535 bytes.
- Never send more than the minimum of `CWND` and `RWND`.
- When congestion is signaled (like with a lost packet), set `ssthresh` to be half of the current window value (or at least 2 segments if the connection is in a *really really* bad shape). During timeouts, set `CWND` to a single segment.
- Use the policies we discussed for Slow Start and Congestion Avoidance to increase `CWND` after each successful ACK.
- Implement *Fast Retransmit*

### Fast Retransmit

Until now, we acknowledged packets based on whether or not they fall within the expected window of the receiver, if a packet out of the window is received, we won't accept it and let the sender timeout and hopefully send it at the right time now, though this scenario rarely happens. What does happen though, is packets being received *out of order*, and that is something that we have not yet really considered.

Of course, our previous TCP still technically works, but imagine this. If the window allows for the sending of 4 segments, our TCP until now sends the 4 segments in a burst and waits for the ACKs.

Let's assume the RTT is well behaved and no `RTO` shenanigans wait for us. If the second segment is lost, our TCP receives the ACK for segments 1, 3 and 4. This increases the window, but since 2 is lost, a timeout is signaled, and we resend from the last unacknowledged segment (i.e. 2) with a smaller window size.

But here is an idea, since the sender is *supposed* to send packets sequentially, the receiver seeing that 2 was skipped can get at least somewhat *suspicious* that the packet might have been lost. Of course it does not necessarily mean that the packet was lost, it just might have experienced some delay. The receiver can now take action to inform the sender that something seems to be wrong. If we can be somewhat sure that the packet was indeed lost and not just delayed, we can ask the sender to retransmit it right away.

This does not help that much on links that have low loads, since the timeout would probably expire before we can actually be sure that the packet was lost, but in really busy links (where the values of `RTTVAR` are actually pretty big), using this method can save a lot of time instead of just waiting for the timer.

#### Implementation

Tahoe implements fast retransmit by waiting for **3 duplicate ACKs**, this is essentially how Tahoe assumes packets get lost other than timeouts. The receiver, once it notes that a sequence number was skipped, keeps sending the ACKs for the previous sequence number instead of the newer ones. Once the sender receives 3 ACKs for the same sequence, it will:
- Set `ssthresh` to half of `CWND`

## Reno

