**Paper:** [A Scalable, Commodity Data Center Network Architecture](http://web.stanford.edu/class/cs244/papers/al-fares-sigcomm08.pdf)

This is a foundational paper about building datacenter-level networks. It is one the main papers that sets forth the idea that at scale, datacenters (DCs) should be built from commodity hardware rather than specialized, high cost hardware (which is contrary to the accepted practice at the time) 

# Background

This paper wasn't produced in a vacuum. At the time, DCs were designed with the purpose of handling large-scale distributed workloads like search engines and Map-Reduce. These:

- Have a very large scale (thousands of servers are involved for each request)
- Require significant **inter-node** bandwidth
- Cause significant **in-cast** (*i.e.,* traffic flooding synchronously into a few (perhaps even a single) node)

For the purpose of cost-saving and ease of management, nodes are often packed into groups of racks called a **pod**. Each pod is often contained and pre-wired in a single shipping container and houses up to a thousand machines, already connected with a Top-of-Rack switch.
The providers deploy these containers, supply power and cooling, and then just like that, they have an extra thousand server capacity available. The only issue here is **how should pods be connected?**.
That is the main question at the heart of this paper (and indeed, any Datacenter design endeavor).

# DC Network Design

A DC network by design, admits a hierarchy. Servers under a ToR switch already have full bisection bandwidth between each-other, and the only question is how to design the "upper-level" network that connects these switches (often called, **edge** switches).

A common approach is a tree-like design:

![[Pasted image 20260907185450.png]]

Whether it is 3-tired like above, or 2-tier by removing the "Aggregation" layer, the network admits two distinct classes of switching hardware:

- Those directly connected to hosts (edge switches/ToR switches)
- Those that merely route traffic between hosts (the switching devices in the Aggregation and Core layers)

>[!FAQ] What About Expanders?
>It is not necessary that we have topologies that respect this distinction. In fact, topologies like **Expanders** exist where each switch must serve and directly connect some $k$ number of hosts and use the rest of its capacity for interconnection. This paper precludes that, we'll handle them in a separate note.
>

The main thing to note is that the switches in the Aggregation and Core layer, by design, require a higher capacity, as they must handle traffic that is begin aggregated from multiple hosts beneath a ToR switch.

>[!FAQ] Why Not Split Uplinks To Multiple Aggregate/Core Nodes?
>That brings with itself routing challenges. The most attractive aspect of the 2/3-tier tree topologies is that routing is trivial!
>That said, as this paper shows, we cannot run away from tackling routing all the time as we are doing here!

So we know how to build such topologies, but the issue here is **cost**.

- Having high-bandwidth switches in the Core/Aggregation layer is extremely expensive.
- Even if we did provide such a topology, most of the time it is wasted. It is very unlikely that *every host* in *every pod* wants to talk to *every other host* in *every other pod*. In other words, providing **full bisection-bandwidth**, *i.e.,* any split of available hosts can communicate arbitrarily with each-other, is often rarely going to be worth it.

Full Bisection-Bandwidth may be need for highly localized traffic (where for example, deeply coupled applications are deployed into the same pod), but between multiple pods, that is rarely the requirements.

Of course, there are outliers (Web Search in particular), but these applications are rare.

For this reason, DC networks are designed with an **over-subscription ratio** in mind. This means that the network is designed to not have full bisection bandwidth for certain workloads in order to make the topology cheaper.

>[!EXAMPLE]
>When we say a network has an over-subscription ratio of $10$, we mean that:
>>**There exists a communication pattern (that is, a set of hosts sending and receiving data in some way) where the lowest throughput that a host experiences is only $10$ percent of its link capacity**.

But even with this in mind, the extreme price jump when going to higher capacity switches, means that the cost will be extremely high even with large over-subscription ratios.

![[Pasted image 20260907194602.png|500]]

The figure above describes the cost of different DC topologies with the number of hosts and over-subscription costs.
As it can be seen, as the number of hosts increase, there is a particular point where there is a sharp increase in price, and that is because we have buy more aggregate/core switches with high capacity.

So the main question is:

> **Can we design a high-performance network that relies only on commodity switches?**

## Clos Networks

Clos network are a family of networks designed by Charles Clos for telephone networks when they had the exact same problem that we are discussing above.

- Telephone networks would rely on "patch-panels" to switch messages. On paper, one would need $N^2$ switch points (or equivalently, $N$ switches with $N$ ports) to switch messages between $N$ hosts in a non-blocking fashion.
- Clos showed that we don't need to go that far, and topologies exist that provide can connect hosts in a non-blocking manner with less than $N^2$ switch nodes. His original paper can be found [here](https://ia601901.us.archive.org/8/items/bstj32-2-406/bstj32-2-406_text.pdf).

>[!FAQ] Clos's Result
>Clos showed that we can create networks with $2s+1$ stages serving $N = n^{s+1}$ endpoints, with switch nodes (or in his words, "cross-points") numbering:
>$$
>C(2s+1) = \frac{n^2 (2n - 1)}{n - 1} \Big[ (5n - 3)(2n - 1)^{s-1} - 2n^s \Big]
>$$

Generally speaking, the more stages we have (*i.e.,* 1, 3, 5 and more), the number of cross-points needed will asymptotically be smaller than with less stages. The tradeoff here is that the more stages we add, the more difficult routing (and even building!) the network becomes.
For endpoints numbering in a few tens-of-thousands, a 5 stage Clos network seems to be acceptable. This 5-stage network can be folded on it self over the 3rd stage, which yields a 3-tier network. This network turns out to be identical to the Fat-Tree!

## Fat-Tree

A Fat-Tree, as mentioned in the previous case, is just a funny way of drawing the 5-stage Clos network, and a slightly easier way of expressing it. We would need $N = n^3$ endpoints, but there is a slightly more appealing way of expressing it. Recall that the number of cross-points in this case would be:
$$
\begin{align*}
C(5) &= \frac{n^2 (2n - 1)}{n - 1} \Big[ (5n - 3)(2n - 1) - 2n^2 \Big] \\
&= \frac{n^2 (2n - 1)}{n - 1} \Big[ 8n^2 - 11 n + 3 \Big] \\
&= \frac{n^2 (2n - 1)}{n - 1} \Big[ 8n(n-1) - 3(n-1)\Big] \\
&= n^2 (2n - 1) (8n - 3)
\end{align*}
$$