**Paper:** Computing Optimal Max-Min Fair Resource Allocation for Elastic Flows

The paper above is an older one, which discusses Max-Min fair TE allocation with elastic flows in mind. This is probably not going to be that useful now, since it is too slow, but it provides a good starting point.

# Introduction

The paper considers Traffic Engineering for *Elastic* flows.
An elastic flow, is a flow that adjusts itself to available bandwidth. This means two things:
- It is delay tolerant. Since if we reduce the bandwidth, the delay naturally goes up.
- It probably has a congestion control protocol like TCP on top of it doing flow/congestion control.

The algorithm presented here, makes a few assumptions:
- **Connections are known in advance**. Essentially, the problem is static. Demands and source/destinations are fixed, and we are asked to route and allocate.
  Understandable, but today, solutions tend to be path-based, which means that only certain routes going out from a node are allowed to be used by any connection.
- **Connections can be split**. Meaning that to route from a source $s$ to destination $d$, we can use multiple paths $p_i$ for routing the same connection if necessary.
  This is not entirely realistic, even for elastic flows. Congestion control protocols do not fair well with out of order packets, and routing over multiple paths heavily increases the chance of out of order packets.
- **Max-Min Fair Allocation Per Link**. In brief, we assume a perfect congestion control, that quickly gives proportional fairness on a single bottleneck link.
  Not a bad assumption by all means ...

# Problem Modeling

Recall that an allocation is max-min fair if:
- No commodity gets allocated more than its demand.
- If there is an unsatisfied commodity, the one with the minimum allocation is maximized among all possible allocations.

Essentially, we want a water-filling like solution.

Now, for our problem, we formulate it as the following:
- A directed graph $G = (N, L)$ is given. Each link $k$ has a capacity $C_k$.
- A set of traffic demands $D$ is given. Each element of $D$ is a tuple like $\langle n_s, n_d, T_d \rangle$ where:
	- $n_s$ and $n_d$ are distinct source and destination nodes picked from $N$.
	- $T_d$ is the total number of connections to be established between the two nodes.
	  This is how we implement multipathing here.

We start with a simpler problem. Assume that all the routes have been assigned. Namely a set of paths $\mathcal{P}$  has been given to us to use. We will use indices $j \in \{1, 2, ..., |\mathcal{P}|\}$ for referencing individual paths. The notation $k \in P_j$ means that the link $k$ is on the path $P_j$.

- let $x_{j,d}$ be the total number of connections routed over path $j$ for demand $d$.
- Let $S_k$ be the maximum amount of bandwidth allocated to individual connections on link $k$.
- A link $k$ is congested if all of its available bandwidth is exhausted. The set $K \subseteq L$ is used to denote the set of all non-congested links.
- Let $f_k$ be the number of connections routed through link $k$

Now, as we hinted, with the routing already set, we can just solve for a max-min fair allocation with water-filling. We also stress that we *assume* that the congestion control is going to give fair assignment to all connections on the same link. This means that we may assume that each connection on link $k$ receives a fair share of $C_k / f_k$  by default.

Now, if routing is set, water-filling can be done quite easily. Essentially, the algorithm would be:
- Find the link with minimum fair share (i.e. find $k$ that minimizes $C_k/f_k$)
- Let $S_k \gets C_k / f_k$ and $K \gets K\;\backslash\;\{k\}$
- Remove all flows crossing link $k$ from the network and the capacity that they are using (this effectively removes link $k$ as well)

We continue this, until we can remove flows no more. If there are flows left in the end, then this routing cannot satisfy the demands, if that is not the case, then we found a feasible assignment given the current routing.

When we do a single iteration of the above, some capacities will change. At the end of any step, all links in $L\;\backslash\;K$ will be fully congested, which means that it must be the case that:
$$
\forall k \in L\;\backslash\;K:\quad C_k = \sum\limits_{k'\;\in\;L \; \backslash \;K} f_{k, k'} S_{k'} + f_k S_k
$$
Where $f_{k, k'}$ is the number of connections that cross both $k$ and $k'$. The reason that we need not bother checking any link in $K$ is that since the rate is limited by the bottleneck, any link that still has residual capacity cannot be the bottleneck by definition and hence has no effect on the above.
