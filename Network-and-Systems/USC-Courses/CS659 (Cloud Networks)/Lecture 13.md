# B4 (Cont.)

We were discussing the TE example from the paper last time. The analysis is on Piazza. The only important things to note:
- Fair share is allocated across all rounds and across all unsatisfied flow groups.
- If a flow group gets satisfied, it is no longer used for calculating the fair share and we can just remove that group as a whole

# The Evolution of B4

**Paper:** [B4 And After](https://research.google/pubs/b4-and-after-managing-hierarchy-partitioning-and-asymmetry-for-availability-and-scale-in-googles-software-defined-wan/)

## Background

B4 scaled heavily, the traffic in particular grew by a factor of 100 and the number of sites also doubled. There is also higher availability requirements these days to maintain competition.

![[Pasted image 20240226161032.png]]

As it can be seen:
- Traffic is growing exponentially
- Topology size is steadily increasing (we'll talk about what domains are)
- The number of Flow Groups and Tunnel Groups are also increasing heavily

>[!Note] Availability SLOs
>The term Service-Level Objective (SLO) usually refers to a number that specifies the (best effort!) guarantee that the network/infrastructure tries to provide. 
>
>For B4, the main concerns are availability SLOs, in particular, a constraint of the form "99.99 percent of the operation time, latency will be less than $l$ and usable bandwidth will be greater than $b$". This "99.99 percent of time" tends to be referred to as a *4-nines* availability.
>
>So basically what happened since the last paper? The SLOs are now much more strict, and we need to make sure we meet them as much as possible.

## Basic Challenges

This paper deals entirely with how a *B4 site* is implemented, in contrast to the routing and TE that we discussed. The problems are:
1. Scaling WAN capacity at a site
2. Address capacity asymmetry
3. Manage switch table size

### Scaling A B4 Site

Remember that the B4 (abstract) topology is made up of some sites that talk BGP with the DCs, and the problem is that the only way that we usually are able to cope with increase in bandwidth demand, is to extend the topology.
The problem with this approach is that:
- It slows down the TE, since the number of paths increases dramatically in a heavily-meshed graph, and thus drops the availability
- It increases switch table size
- Complicates capacity planning and management

The solution here is quite similar to the ideas that were used for Jupiter and Saturn.

![[Pasted image 20240226163229.png|300]]

Each Saturn site would be a 2-level Clos topology. The lower level is deployed fully with 4 Saturn chassis, but the upper level is expanded incrementally.
As technology marched on, similar to what happened with Saturn itself, the original B4 sites evolved into the *Stargate Sites* currently used:

![[Pasted image 20240226163839.png|300]]

A Stargate site is at most a 4 node mesh of Stargate *Supernodes*, which unlike in Jupiter, they did not need freedomes. Each Supernode can serve multiple DCs and is created from 32x40 Gbps chips.

The bigger change though was the introduction of *Side-Links*.

#### Capacity Asymmetry

One problem that needs to be resolved happens in a scenario similar to the following:

![[Pasted image 20240226164806.png|400]]

Now, in the original B4 site, it was possible that at most 4 nodes can exist within the site, but the topology abstraction required for TE scaling would abstract away all connection between the nodes in a site. Thus, in the site above, within $A$, the TE won't have any knowledge about the links between $A_1$ and $A_2$, so if a failure drops the total site to site capacity from $A$ to $B$, the TE algorithm won't be able to make use of the intra-site links, so it settles on a sub-optimal value.

In the example above, when the two yellow links drop their capacities from 5 to 1 due to say some fiber failure, at most 2 units of traffic can be pushed through $A_2$, and thus the capacity would settle on 4 instead. On the other hand, if either the global TE could make use of the link between $A_1$ and $A_2$ ***OR*** an algorithm could run within the site itself, then things could be much better (the result would be 12 here for example).

This link between $A_1$ and $A_2$ are known as Side-Links, and the only way we could effectively make use of them, is to use hierarchical TE algorithms, one on the super-node level and one on the global topology. This is complicated, but worth it, since it helps achieve availability. 

>[!FAQ] An Assumption on The Link Failure Rates
>It is important to note that Google, currently, does not seem to implement any coordination between the two TE levels, that is just too complicated. 
>Thus, their algorithm does *NOT* cope well with failures of the side-links themselves!
>
>The paper mentions that the side-links are much less likely to fail compared to the site-to-site links and thus they are not too worried about this. Note that this whole site exists in a DC!

#### Hierarchical TE Implementation

As we remember from the TE implementation in the last lecture, since traffic can not flow through a non-shortest path, then tunneling is needed, which can be implemented using IP-in-IP encapsulation.

![[Pasted image 20240226171302.png|500]]

Again, noting that the picture above is the ORIGINAL B4, not the new one. The new TE implementation requires double IP encapsulation, since there is two TE stages, but the problem is that with this two level encapsulation, the hashing algorithms that are used for implementing multi-path routing or load balancing become much less effective.

We should also mention that the paper explicitly mentions that single level TE at both site and global level runs **200 times slower** according to the paper, it's unacceptable.

So the challenge is, "can we implement a tunneling mechanism that does not use 2-level encapsulation but still splits the traffic arbitrarily?"

>[!IMPORTANT]
>As you have seen, the traffic is always split equally among the egress site super-nodes. This is necessary to make sure that the global TE won't have to know anything about the site level TE.

For this reason, Google extended their previous notion of a Tunnel Group to a *Tunnel Split Group (TSG)*. A TSG looks like a single Tunnel Group when you inspect it between sites, but within them, the tunnel can split traffic between available links.

![[Pasted image 20240226172217.png|500]]

It is worth diving into these algorithms later, but in brief:
- The TSG problem can be converted to a network flow problem on a graph $G_{TSG}$ over the site topology. For routing reasons, this graph MUST be a DAG for each TSG.
- The solution to the above problem would be a series of data entry outputs that can be pushed into the switch, and thus, will require sequencing to make sure they are hitless. This sequencing is simple here, as it can be done by backtracking through the DAG.

"Professor skipped the Switch Split Groups slide (i.e. slide 39)"

### Minimizing Switch Table Size

Switches have different tables, not just for routing! There are the good old Longest Prefix Match (LPM) which are stored on the SRAM. There is the ACL (pronounced ACKEL by the way!) which lives on the TCAM (so power hungry and expensive), and then there is the ECMP/WCMP table for the ECMP groups and their buckets and weights if WCMP is enabled.

The problem was that B4 grew so large that the tables actually had to drop entries! This means that:
- FG to TG matchings that live in the ACL on the TCAM, exceeded the limited available TCAM capacity.
- WCMP splitting even started to exceed the available SRAM, which was thought to be almost impossible!

The problem stems from the diversifying QoS implementations that Google was using. If the ACL were to just match prefixes to Tunnel Groups as we did originally, things would be fine, but currently, the ACL actually matched the QoS **AND** the prefix, and that scaled up the size of the ACL too much.

Google used a classic solution, Virtual Routing Functions (VRFs), which are essentially ACLs that live in the SRAM instead. So each QoS gets its own VRF in the LPM and only prefix to TG matching goes in the TCAM.
Note that QoS is matched via IP fields (e.g. DSCP).

# SWAN

Already discussed the [[SWAN]] paper before! The main method to consider here is *demand shifting*, so shift background traffic (or elastic traffic) to when there is more capacity.

>[!TODO]
>Read the [MPLS primer](https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=826369)!

