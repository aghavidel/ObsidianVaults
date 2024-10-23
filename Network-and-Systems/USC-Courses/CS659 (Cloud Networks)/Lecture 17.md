# Swift (review)

The Swift paper is about congestion control in the DCN with delay as feedback instead of packet loss.  Swift notes that DCNs need very low tail latency (or at least predictable latency).

The reason for this heavy limit is that most DCNs run a distributed file system, where *every single disk access* goes over the network. If the network takes several milliseconds to respond to these, then it is already a bottleneck! We really don't want that.

![[Pasted image 20240318160619.png]]

These storage systems are also the majority of the traffic in the entire DCN fleet, so we want to put tight bounds on delay.
There is also a matter of good historical timing. This paper came after DCTCP, and it was the time where kernel bypass networking was becoming a trend.

![[Pasted image 20240318161040.png|500]]

Kernel bypass has many benefits:
- Allows NIC-level timestamping
- Careful packet *pacing*
- Reduces CPU usage *significantly*

Swift also does not assume switch support, so no ECN as well. Swift did not use TCP, or the socket API for that matter, they implemented a congestion control protocol from scratch and like DXTCP, it is also **designed exactly for DCN usage**.

## Swift Approach

Swift is **Delay-Based**, which means:
- Measure end-to-end RTT
- Decompose RTTs to host and network contributions 
- Use AIMD to converge to a target delay
- Respond differently to host or network congestion

This is the main idea of Swift, figuring out what exactly contributes to each part of the delay:

![[Pasted image 20240318161619.png]]

The green parts are because of hosts, and the red parts are because of the network.

>[!IMPORTANT]
>Note that the NICs can timestamp each packet on ingress/egress (most NICs can do that, even today). Of course to make use of these timestamps, hosts need to be synchronized. 
>
>![[Pasted image 20240318162453.png|500]]

## Swift Algorithm

Assume we have a function `TargetDelay()` that *somehow* gives the target value of delay required. The CCA would do the following:
- If RTT is lower than target delay, then do additive increase. Do this per packet (NOT like DCTCP).
- If RTT is higher, then do multiplicative decrease. Do this **once per RTT** (just like DCTCP).

One important difference also is that each RTT, if we do have to decrease, we factor in how much we are far from the target RTT:
$$
\text{cwnd} \gets \max(1 - \beta.(\frac{delay - target}{delay}), 1 - \beta_{max}) . \text{cwnd}
$$
For increasing, just simple additive increase:
$$
\text{cwnd} \gets \text{cwnd} + \frac{\alpha}{\text{cwnd}}.\#Acks
$$
As with most CCA schemes, instantaneous RTT is considered when deciding how close/far we are from the target delays.

So where do we use the fact that we have different contributions from network and hosts?
- Keep two different congestion windows internally. One for the network (the *fabric* congestion window, `cwnd_fabric`) and one for the endpoint (the *host* congestion window, `cwnd_host`).
- At each round, let $\text{cwnd} = \min(\text{cwnd}_{\text{fabric}}, \text{cwnd}_{\text{host}})$
- Endpoint and Fabric delays **also have separate target delays**, which can be tuned based on the applications if needed.

## Incast Problem

When thousands of network workers respond to aggregator at the same time, we have *[[Incast]]*. This is bad for shallow-buffer switches, which may not even have space for one packet (which means that congestion window can be less than 1!). Many CCAs do not consider these trouble makers.

To signal this, Swift makes use of this same insight. To accommodate large Incast, let the congestion window be less than 1 and pace the packets. The paper discusses why pacing is good (which is not surprising, since [[BBR]] also paces packets, but BBR does it for **all** packets).

Swift does this only when it detects Incast. If $\text{cwnd} \lt 1$, then:
$$
\text{cwnd} \gets \text{cwnd} + \alpha.\#Acks
$$
Swift will pace packets in such a way that for each RTT, we send only a $\text{cwnd}$ worth of packets. Swift does not do pacing for all packets, since that has a noticeable cost in CPU.

## Target Delays

How to measure the target delays?
It depends on many things. First, note that delays in the network have 3 parts:
- *Serialization Delay*, or transmission delay, that depends on the  NIC bandwidth
- *Propagation Delay*, time needed for wave propagation in the transmission medium
- *Queuing Delay*, which we are quite familiar with, and quite afraid of

If the queueing delay is low, which can only happen if the CCA does its job correctly, then we would have:
$$
d_{target} = d_{\text{per-hop}} . \#Hops + d_{base}
$$
Here, $d_{base}$ is the propagation delay and $d_{per-hop}$ is the same delay per hop. We can observe the number of hops by checking the TTL on received packets.
This is nice enough for a single flow, but for multiple flows, things become complicated, mostly because queuing delays become significant. 

![[Pasted image 20240318172250.png|500]]

From above, you can see that the queue length (and hence the delay), behaves as $O(\sqrt{N})$ for $N$ flows. Taking this into account would mean that the target would have another term added to it. The details are involved, but we won't discuss that with detail.

## Performance

![[Pasted image 20240318172928.png]]

Here, GCN is essentially just DCTCP.

# Network Virtualization in DCNs

In general, *virtualizing* any resource $X$ would create a series of resources $X_i$ such that:
- Only $X$ provides for all $X_i$ resources.
- All pairs $X_i$ and $X_j$ are completely independent and isolated from each other.
- $X_i$ cannot have any effect on $X_j$ performance.

This gives the illusion of physical resources to whatever will utilize the virtualized resources. 

Networks can also be virtualized, but their virtualization is a bit more colorful. 

![[Pasted image 20240318173737.png|300]]

A network can be virtualized so that it gives the illusion that it provides a certain topology to a series of hosts. They can even provide links that do not even exist, as you can see in the picture above, where red and blue networks are virtualized over a physical network. These networks  are independent of each other, and no host in the blue network can actually reach the red ones. 

This idea is useful, and not particularly new. It was envisioned at least when people were coming up with VLANs. 