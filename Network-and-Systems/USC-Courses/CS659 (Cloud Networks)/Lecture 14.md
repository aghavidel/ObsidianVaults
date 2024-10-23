
# SWAN

The paper intro is already in [[SWAN]], so look into it if needed.
First some background:

>[!REMINDER]- MPLS Primer
>It has been some time since we seen MPLS again, so let's review some things.
>MPLS encapsulates normal layer 2 packets and are used for tunneling.
>
>![[Pasted image 20240228160635.png|300]]
>
>The main thing about MPLS that makes it attractive (somewhat!) is that it is faster than IP forwarding between hops, since MPLS headers used to switch them are smaller and much faster for matching.
>
>As for how this can be used for tunneling:
>
>![[Pasted image 20240228160833.png|400]]
>
>To implement an MPLS fabric, the network uses *Label Switching Routers* (LSRs) to forward packets on *Label Switched Paths* (LSPs). These routes effectively act as tunnels, since LSRs **do not look at the IP header if there is an MPLS header**, thus from the perspective of the ingress IP switch, there is just a single observable hop to the next egress IP switch, whereas a whole network can possibly sit in between them.
>Optionally, a router can change the label, add a new one, or just strip it to resume normal IP routing.
>
>Since LSRs effectively use MPLS labels to determine the next hop, they are handy for traffic engineering, since they can override the typical shortest path routing.

>[!FAQ]
>Microsoft's WANs already had MPLS routing enabled into them (they are primarily made out of Cisco routers), thus it was quite natural for them to gravitate to this method. In contrast, Google, which used Merchant Silicon for their networks, did not have this technology enabled into their routers and thus used IP-in-IP tunneling.

## MPLS-TE Intro.

Consider the following network:

![[Pasted image 20240228161727.png|500]]

Note that MPLS very much predates SDN, and thus, whatever method used for both setting up the paths and signaling them to routers, a distributed algorithm was used.
The algorithm does not really matter, you can assume it is just something similar to Google's solution.
The operator will also specify its own set of *constraints* that will go through algorithm. In the end, the IGP will announce the available bandwidth on each link and them advertise them to routers. 

The signaling of these paths to routers is done with RSVP, and as part of this setup, the label switching tables are also setup within each router.

This method can:
- Split traffic to the same destination on different routes
- Split traffic between a pair of nodes along different paths

### Downside of MPLS-TE

It quite simply isn't optimal. It tends to greedily assign traffic to the shortest tunnel that has available capacity. Here is an example:

![[Pasted image 20240228163340.png|500]]

>[!EXAMPLE] 
>Take the above example.
>Here is how requests come in:
>- Flow 1 wants a path from $R_1$ to $R_6$
>- Flow 2 wants a path from $R_3$ to $R_6$
>- Flow 3 wants a path from $R_4$ to $R_6$
>
>If you are greedy, and choose $R_1 \rightarrow R_2 \rightarrow R_6$ for the first flow instead of $R_1 \rightarrow R_7 \rightarrow R_6$, then you would have a suboptimal solution.
>If you knew in advance the whole set of flows, or could modify them as a centralized entity, then this could be prevented.

Beyond this, there is also a concern about fairness, since not everyone would want the same capacity. It is not obvious how (if at all) one would be able to implement a max-min fair algorithm on this topology with MPLS-TE. The main reason is that MPLS allocates capacity in proportion to the demand, so you can have a case like the following:

![[Pasted image 20240228164130.png|500]]

An SDN approach can prevent these from happening.

>[!REMINDER] Max-Min Fairness
>In case you forgot, Max-Min Fair allocation is any allocation where:
>- No one gets more than their demand
>- If there are unsatisfied allocation, the minimum allocation in that set is maximized
## SWAN Design

In SWAN, traffic is prioritized into 3 classes:
- Interactive
- Elastic
- Background

This is from high to low priority. Traffic that goes into the same class must satisfy fair sharing. 
The problem here is a computational one, but here, they attempt to scale it down with a fast-approximate algorithm.

SWAN also makes note that any update into the network routing needs to be *sequenced* to make sure that they are hitless and don't cause packet drops. One could also be worried about preventing congestion too!
The paper gives this example:

![[Pasted image 20240228164938.png|500]]

Here, "a" is the initial state and "b" is the desired state.
Another challenge here is also the fact that the table used for these operations could get too large to fit in a switch, so they need to be scaled down as well. The main idea for this is to be lazy and populate the tables *On-Demand.*

## SWAN Implementation

SWAN is quite similar to what Google does, there are 3 main components:
- **Hosts:** These are servers that track service demands, report them to the Broker node and do rate-limiting on behalf of it.
- **Broker:** This is essentially the same as Google's Bandwidth Enforcer. It sets host bandwidths and aggregates their collected results for the controller.
- **Network Agents:** These convey topology state to the controller. They are essentially the same as Gateways in Google's design.

![[Pasted image 20240228165821.png|600]]

The main challenges here are:
- How to compute fair allocations quickly
- How to install them in a congestion-free manner
- How to fit the resulting tables into a router

The linear program that SWAN implements is essentially the same one that we discussed in the context of the Jupiter paper:

![[Pasted image 20240228170518.png|500]]

### Basic SWAN Algorithm

The allocation algorithm iterates on all links. As mentioned, it prioritizes based on traffic class (Interactive > Elastic > Background). Based on the previous notation, at each step, the allocation will:
$$
{b_i} \gets \text{Throughput Maximization Approx. Max-Min Fairness}\;(Pri, \{c_i^{remain}\})
$$
The approximate algorithm uses a Multi-Commodity Flow program:

![[Pasted image 20240228171310.png|500]]

Solving this MCF repeatedly on different input matrixes takes a very long time and is just too inefficient. So SWAN uses an *approximation* for this algorithm instead. 

![[Pasted image 20240228173435.png|500]]

The main idea here is that we can trade-off fairness and runtime using $\alpha$ and $U$.
- A large $\alpha$ would make large geometric partitions and thus lower the runtime but make it less fair
- A smaller $\alpha$ would make it more fair but it would take much longer

This is where this approach is much more grounded than Google's, since their approach greedily fills the shortest path tunnel and is quite heuristic. SWAN on the other hand gives clear guarantees on what it is actually achieving! The guarantee here is that any allocation $b_i$ would be within $\alpha$ of its max-min fair allocation value.

### Making Smaller Tables

Routers have small, and not to mention variable table sizes, and these are very hard to encode into the optimization algorithm, and the paper also mentions that adding these constrains into the algorithm, it becomes just too complicated. 
Their approach is actually pretty smart, they would *refine* the output of the algorithm, they:
- Select every lowest-latency tunnel between DC pairs and greedily select maximum-traffic tunnels until the tables are full.
- These tunnel are then removed from the problem consideration and the optimization is ran again on this result

This can mean that the final result might show lower utilization but it is still max-min fair.

### In-Place Updates

To make sure that updates are congestion free, another optimization is solved that would spit out a series of intermediate configurations that are congestion-free.

![[Pasted image 20240228174800.png|500]]

This can only be done if some **spare capacity is reserved**. If a fraction $s$ is reserved, the algorithm can find configurations sequences of length less than or equal to $\lceil \frac{1}{s} \rceil - 1$ .

>[!THEOREM]
>If $s=0$ (i.e. no spare capacity), then congestion-free updates are impossible.

Beyond just link capacity, we also need spare *Table Capacity* as well, which we call **Scratch Capacity**
A similar result compared to above can be proved for this scenario as well. If we have a spare percentage of $\lambda$ for scratch capacity, then we can sequence tunnel set updates in $\lceil \frac{1}{\lambda} \rceil$.

All of these optimizations put together, we get this:

![[Pasted image 20240304162513.png|500]]

B4 and SWAN represent the state of the art for doing TE on large, global DC scale. It is a very impressive technological achievement. 