# DCTCP

The focus of this paper is to change TCP to make it better suited for DCs.

## Background

The main thing that derived DCs was the invention of Broadcom merchant silicon chips that were much cheaper than the Cisco chassis that had dominated the market for decades. The Cisco chassis were an order of magnitude more expensive than the Broadcom chips, yet they had better hardware.

The big downside with Broadcom was that you had to implement your networking stack yourself. No problem! Hyperscalers have whole armies of engineers to make them, but the big thing that kept getting in the way was *Memory*.

The main reason Broadcom chips were cheap in the first place was really because they didn't have as much memory as Cisco did, and while that trade off is nice, we still need to find some way to remedy it. This is where TCP congestion starts to become a big problem.

>[!REMINDER]- Explicit Congestion Notification (ECN)
>The IPv4/v6 standard allows a router to set a bit in the IP header called the ECN bit that would notify a sender of an immediate congestion in the upstream.
>When a sender receives this, it will halve the TCP window and let the congestion wear out.
>
>![[Pasted image 20240304163543.png|500]]
>
>Usually, the algorithm is implemented like this:
>1. A switch marks a packet with an `ECN` flag if it sees that the queue length exceeded a threshold. Other policies can be used as well.
>2. When the destination receives that packet, it will set the `ECN` flag on the ACK of that packet and send it back to the source.
>3. The source, upon receiving the this ACK, should usually send another ACK with a special `CWR` bit set, which conveys to the destination that the ECN was received.
>   The source will then decide what to do with the knowledge that the packet saw congestion (e.g. chop the window in half!)

So how can memory interfere with TCP congestion control?
Well, the main model that the DC operates on is the *partition-aggregator* model:
- Workers perform computations
- Aggregators receive and "patch" worker results together

![[Pasted image 20240304163744.png|500]]

This is heavily utilized for GPU clusters and things like WWW search and much more.
To control latency, all point to point connections between workers and aggregators (or aggregators to aggregators) have a *soft deadline*. If this deadline is exceeded, the result of that connection is not reflected in the returned query to the user.

For example, with Search, this can produce less results than expected or lower quality results.

Now, the characteristic of these traffics is that:
- Aggregate to worker traffic:
	- Generally smaller transfers
	- Can be synchronized, and thus become very bursty
- Aggregate to Aggregate traffic:
	- More stable, less bursty, but can occasionally get very very large

![[Pasted image 20240304165010.png|600]]

The key takeaway from the figure above is that:
- A significant portion of total traffic is from very few very large flows (*elephants*)
- The majority of the connections are short lived and need not much bandwidth (*mice*)
- **There is no statistical multiplexing**

<u>We want high throughput for elephants and low latency for mice. As for any CCA, we also need large burst tolerance.</u>
## Effect of Switches

Since Merchant Silicon does not have a particularly large memory and even that small amount should be shared among multiple connections. We once again have the reverse buffer-bloat problem (also called buffer-pressure, where large flows can shut-off smaller flows). Letting TCP timeout by filling the buffers would have very bad performance.

Note that this paper predates BBR, but it is trying to solve the same issue that BBR solved.

There is also another problem, much known in the DC world, the problem of **[[Incast]]**.

![[Pasted image 20240304170937.png|500]]

In the partition-aggregate architecture, multiple queries to workers can happen at the same time, and with the nice DC networks, the responses come close together as well. These synchronized responses come through the *same* switch and use a massive amount of memory. 

>[!NOTE] Incast Solutions With Pure TCP
>Incast is a well-known problem and there has been significant work done to resolve it. The two popular ones were the following:
>
>1. **Jitter Worker Responses:** To smooth out the synchronization, we forcefully de-synchronize by doing small random back-offs (remember for example in ALOHA or Wi-Fi). This helps, but increases the median response time.
>   
>   ![[Pasted image 20240304171640.png|500]]
>   
>2. **Reduce the minimum RTO timer of TCP**: Empirically, this has not yielded acceptable results.


## DCTCP Implementation

We want a TCP implementation that:
- Works with shallow-buffered switches
- Creates small queues
- Prevent Incast traffic

For DCTCP, the main idea is that when a router sees that the queues have gone above a certain value, the router will send an ECN to the source.
The main difference here is how the ECN is treated. Instead of chopping the window in half, the window is decreased in proportion to the fraction of ECN packets received. This is important, since it prevents big drops in throughput without reason. 

So this is a modification to the multiplicative decrease of typical TCP. The decrease factor is scaled with the number of ECNs received and is reset when a whole window is successfully transmitted. 

### Marking At The Router
```c
// Marking
if (instantaneousQueueSize > K) {
	packet->ce_bit = 1;
}
```
Take note of the fact that *instantaneous* queue size is used. In RED for example, we used average queue size, this is important and we will see why soon.

### Conveying CE Signal To Receiver

We could send CE signal on every ACK, but that is ideal, since TCP uses delayed ACKs most of the time, where we ACK after getting some amount of packets first.

![[Pasted image 20240304174801.png|500]]

The state corresponds to the `ECN` bit on that *last received packet*.
What this means is that:
- While `ECN` is not seen, ACK each $m$ packet cumulatively
- The moment `ECN` is seen, send an immediate ACK with `ECN` set to 0
- After this, send an ACK with `ECN` set to 1 for every $m$ packet
- If `ECN` is not seen, revert back and send an immediate ACK with `ECN` set to 1

So for example, with typical $m=2$, if the sequence of packet `ECN` seen is `0 0 1 0` and no congestion was sensed before, then the `ECN` flag on the ACKs sent would be `0 1 0`, where the first one ACKs both of the first 2 packets, the second one ACKs the congested packet, and the final one ACKs the one without congestion that came after it.

The receiver would see from the sequence numbers on the ACKs that the first two packets saw no congestion, but the third one saw a large queue, but the packet directly after it did not.