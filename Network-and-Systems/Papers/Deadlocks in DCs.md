**Paper:**
- [Deadlocks in Datacenter Networks: Why Do They Form, and How to Avoid Them](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/11/rdmahotnets16.pdf)
- [RDMA over Commodity Ethernet at Scale](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/11/rdma_sigcomm2016.pdf)

This paper describes a phenomenon that started to re-emerge within large-scale data center networks as they started to adopt RDMA. In particular, this paper focuses on [[RoCE]], which has found much usage due to its innate compatibility with already-existing IP fabric (in contrast to [[InfiniBand]]).

# Intro.

Operators have been deploying RDMA for a long time now, with [[RoCE]] being one of the top choices in face of other competitors (in the [[IRN]] paper, it is described what the competition looks like and who is favored for what reason) .
RDMA, aside from its different addressing API which focuses on memory mappings rather than byte streams, is efficient due to the fact that it has much less logic built into it. The original RDMA specification in [[InfiniBand]] considered very pristine networks where packet loss happens only in the case of in-cast or congestion. In such a network, packet loss rarely happens if the network is correctly tuned, but when it *does happen*, we really feel the hit.

For this reason, RoCE in particular has focused heavily on keeping the network fabric basically lossless, in the sense that the network tries to never let switch/port buffers reach a point where congestion becomes unavoidable. Many of these networks also take advantage of the fact that they are with high probability operating within a data-center, where the important characteristics of the network (like say, the bandwidth-delay-product) are known and do not change.

In such context, simple methods of flow control might be better, and PFC seems to be one of the better methods among that set (see the [[RoCE]] document for more discussion on PFC).

## Problems with PFC

PFC creates a lossless L2 network (and thus, makes the network effectively lossless) by preventing buffer overflow on the receiver end. This is done as follows:

- Over a connection established between the sender (upstream) and receiver (downstream), the receivers track the buffer size.
- If the buffer size exceeds some pre-configured amount, the receiver sends a notification to the upstream (this is referred to as a `X-OFF` message in Ethernet spec.).
- The upstream, upon receiving `X-OFF`, stops sending new packets. Once the buffer size on downstream goes below another pre-configured threshold, a `X-ON` message is sent and unblocks the upstream, resuming communication.

It is easy to see that as long as the stop threshold is less than the buffer size minus the BDP, this will prevent any packet loss due to buffer overflow. The resume threshold obviously must be some number strictly less than the stop threshold, but some other considerations must go into tuning it (for example, if the two are too close, you can get flapping between the blocked and unblocked state, which might be undesirable), but if the two are too far apart, the network might remain underutilized.

As simple as PFC is, it brings with itself a whole host of issues, which is the main pain point that we want to expand on. To get a better idea of this, we need to focus on how RoCE has been deployed over the year. The reference we will be using is somewhat outdated (it is a decade old!) but it is pretty well written and worth a read.

## Deployment of RoCE in Modern DCNs

Throughout this document, "RoCE" means "RoCEv2", or RMDA over IP fabric which is the de-facto RDMA deployment for intra-DC communication. The reason that RDMA has effectively replaced TCP in this context is twofold:

1. Sending (and especially receiving) TCP packets at a high rate is very CPU bound, even with the numerous kernel offloading methods that have been put forward (which merits its own documentation in the future, if the author isn't lazy).
2. Many DC applications like web search are *very* delay sensitive (we discussed this with DCTCP in [[USC-Courses/CS656 (Advanced OS)/Lecture 15|Lecture 15]]). TCP might eventually achieve good throughput but it almost always incurs quite a bit of delay for multiple reasons, chief among these are a) Kernel latency is always present, sometimes as high as tens of milliseconds b) Packet loss, even in DCNs, still happen, and that is because traffic is always bursty and buffer overflow can happen. In such cases, TCP needs to timeout or fast retransmit in the best of case, and latency still spikes up because of that.

RoCE over RDMA works by encapsulating Ethernet frames inside typical UDP packets. In Microsoft DCNs, these packets are tagged per Queue Pair (QP), where:
- Destination UDP port is usually fixed (the paper says port 4791 is used)
- Source UDP port is randomly selected per QP
Intermediate switches use ECMP, with standard hashing. This means that for each QP, the route taken is fixed, minimizing the possibility of having to handle out-of-order packets on the receiver. For different QPs, routes are distributed among all available choices.

RDMA, by nature, is unable to handle cases where packet loss due to buffer occupancy happens. To be more precise, RDMA caters to cases where it is *feasible* to assume such things can be done away with, in order to maximize throughput and minimize latency.
In RDMA, packets still have checksum and corruption can be reported per-packet, but packet loss due to buffer occupancy is not acceptable. Obviously, rare cases can happen (for example, PFC might be tuned incorrectly or the NIC might be noisy), but these are most definitely not the normal operation regime of RDMA NICs.

To provide lossless fabric, traditional RoCE uses PFC. One of the very well known issues that PFC causes is *Head-of-Line (HoL) Blocking*. This is because pausing the whole upstream from sending is too coarse-grained.
- A single sender is very likely multiplexing multiple flows over the network. Some of these flows might be heavy-hitters and contributing to the buffer occupancy, but some (in fact many of them in practice) are very short lived, having a very small buffer footprint.
- PFC however pauses *all* of these. Even if one of the many flows is the prime contributor to buffer occupancy, PFC will pause all of these flows, including the in-elastic, short-lived flows that are much more sensitive to delays.
For this reason, PFC incorporates priority queuing, where buffer residency is tracked over a set of predefined priority classes (which is a small number, say 8 ...), and PFC pauses only upstream queue of the associated priority class.

![[Pasted image 20260218223458.png]]

This figure represents the essence of the procedure. `p1` is the queue with the highest receiver buffer occupancy, and as such, PFC only flags flows that fall into that particular priority class.

This helps alleviate HoL blocking, but in practice, it works worse than expected, because the number of priorities in practice turns out to be too little. Note that no matter how fine-grained the priorities are chosen, each queue still must maintain at least a BDP worth of headroom to account for the delay of each hop.
In Microsoft, Spine switches have a hop length of around 300 meters. At 40 Gbps, this translates to a BDP of around `10.7 KB` if the network looks like the following:

![[Pasted image 20260219125058.png|400]]

With this amount of BDP, and the fact that the switches (at least at time of writing) were shallow-buffered switches with a single 9-12 MB pool for all packets, basically meant that among the 8 priority classes, only 2 were usable. One class has thus been assigned for latency sensitive, inelastic flows and one for bulk transfer, assuring that PFC HoL blocking does not effect these two at the same time.

Priority can be tagged on packets either using VLAN or DSCP (the latter is preferred in the Microsoft paper). 

## PFC Livelock

>[!NOTE]
>I have issues with the statement that "PFC can cause livelocks". The issue discussed here in the paper is all about the retransmission scheme rather than PFC. In fact, it would apply to any CCA.

Livelock can happen if the RDMA fabric uses Go-Back-0 for retransmission. In particular, the case where the *entire* transmission will be repeated if there is even a single packet loss. This is *obviously* quite stupid but it has the benefit of abstracting away a lot of the extra state that would have to be maintained if CCA is needed.

The lowest hanging fruit is to do Go-Back-N. I wonder if someone decides that is not enough and writes a paper on it ...

## PFC Deadlock

Now the more interesting stuff, deadlocks!

![[Pasted image 20260219162020.png]]

The attached figure describes a deadlock in a *Clos* network that Microsoft actually saw happen. The mechanism is quite convoluted, so the figure above should be noted while following the description.

>[!REMINDER]- L2 Forwarding
>ToR switches usually do simple L2 forwarding among all the servers underneath them. This means that they just look for the destination MAC address and forward on the port they last heard that MAC address sending things from. In the case that the mapping is missing, they must *flood*, which means that all servers connected to the ToR will hear about the packet, and any port that is not the designated receiver must drop the packet once it sees the mismatch between the packet and the actual destination MAC address.

First, note that deadlocks are in general not expected in a Clos topology, since routing by definition prohibits loops (packets can only go strictly up or down with a single direction change in the routing hierarchy, which constitutes what people call "Valley-Free" routing).
