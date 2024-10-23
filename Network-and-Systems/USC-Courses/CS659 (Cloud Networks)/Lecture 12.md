
# B4 (Cont.)

We will continue our discussion of B4, and we begin by discussing B4 on a high level:

![[Pasted image 20240221160831.png|500]]

We should remind you that the word *site* tends to take many different meanings, but here it essentially corresponds to the blue rectangles that we saw on the B4 global map. Each site will serve a cluster (imagine like a Jupiter DC).

The B4 WAN as a whole is a single AS, and each site talks eBGP with Jupiter DCs. The B4 network of course would have to run iBGP sessions between each other for consistency, however, Google *Also* runs IGPs between B4 nodes, namely, they run IS-IS sessions, and we will see why they chose this.

This describes the data plane, but the control plane also is globally distributed among the clusters. Each B4 switch runs an OpenFlow Agent (OFA), and OpenFlow controllers like Orion instances are deployed to control them.
These controller are replicated and run distributed leader election algorithms with Paxos. Each controller runs on a Network Controller Node (NCS), which is just a VM, running on a rack somewhere (presumably close to the physical site of the B4 node).

Now, how do we route this creature?
Typical shortest path schemes can certainly be used with IGPs, and that definitely works, but there are 2 problems with all shortest-path routing schemes:
- They achieve low infrastructure utilization
- They are demand-oblivious, and thus, result in congestion

So why deploy IS-IS in the first place?
Two reasons:
- You need IS-IS to announce TE data across the network, and the controller will need it for optimizing the network
- You also need it for a fallback. You should note that B4 is the *first* SDN WAN ever, and so, Google was not sure how it would turn out. This is very much like with Firehose 1.1 and the 4-post topology, where the new infrastructure was deployed as a bag on the side, and could fallback on the previous infrastructure if things got really out of hand.

All SDN controllers are then logically connected via a global *Gateway* node (it is not specified whether or not this network is in-band or out-of-band), and through this gateway, they connect to a global TE server that actually does the TE computation for the whole network.

Both the gateway and the TE server are replicated. So this is a hierarchical SDN design.
- Each controller instance will assume control of the 4 routers in each B4 node. Later, Orion would take over these.
- Each controller runs at least 2 applications:
	- BGP/IS-IS implementations (Like Quagga, and Google's own implementation, RAP). Orion would later combine these two implementations into *Raven*, which means that Orion can now implement BGP as both a centralized instance and a legacy distributed version.
	- A TE implementation and optimizer (TE Agent). This is separate from Orion, but most likely would have to interact with the Orion Routing Engine
- Controllers are replicated over 3 servers, and Paxos is employed for leader election

![[Pasted image 20240221164021.png|500]]

Controllers will install rules on switches to send all BGP updates as Packet-In messages to the controllers. When a centralized execution is in process, the controller consumes the message and uses RAP for updates. If decentralized execution is needed, the controller will relinquish most of its opperation to legacy software like Quagga.

## TE Implementation

### Problem Formulation

A TE problem, is fundamentally an optimization problem, where we receive a topology and a node-to-node demand and are asked to produce flow assignments. Usually, the objective would be to satisfy the demands as much as possible while preserving max-min fairness.

The topology is very abstract, a whole site is shown as just a single node, and the reason for this is that it helps the optimization scale a lot more. Google also introduces a notion of *Flow Group*, which is a traffic abstraction that we will discuss.

![[Pasted image 20240221164508.png|200]]

Take the above topology or example.
- Each node is a B4 site, which is actually an abstraction of many nodes. This is how Topology Abstraction is realized.
- As for Traffic Abstraction, imagine we have multiple flows from B to A, $J_i$ for $i \in \{1, 2, 3, ..., n\}$
	- Instead of feeding all $n$ flows as individual inputs to the optimization, the problem groups these flows together according to a policy $\Pi$. So if $\Pi(J_i) = \Pi(J_j)$, then the two flows are grouped together as a single one. This of course partitions the set into sub-groups which we call Flow Groups.
	- Usually, $\Pi$ takes the traffic QoS into consideration, and maybe some header data. The paper does not discuss this policy assignment in much detail.

Phrasing the problem in this way, it generates a demand matrix $D$, where $D_{ij}$ would be the aggregate group demand from node $i$ to node $j$. Given a capacity matrix $C$, we can solve this optimization problem as we see fit.

The output of this optimization is formulated as a *Tunnel Group*, which is a path that routes all flows based on the output of $\Pi(J_i)$.

### TE Enabled Infrastructure

![[Pasted image 20240221170009.png|500]]

There are multiple pieces at play here.
- The SDN gateway connects to the NIB instances of each controller and consumes topology and routing events. The gateway routes these updates to the TE Server.
- The TE Server, consumes the updates from the gateway in the Topology Aggregator. The aggregator employs a policy based mechanism by Google itself to abstract the current global topology into the smaller one that we saw above for example (so the abstraction is dynamic, not static)
- There is also a piece of software called **Bandwidth Enforcer**:
	- It has its own paper, so it is probably too complicated to discuss right now 
	- Basically, it is a distributed software that runs on *ALL* servers deployed by Google, and it measure the traffic input and output to the server and sends it to the controllers.
	- The Enforcer will calculate the Flow Groups using Google's specified $\Pi$, and feeds it to the TE Server.
- The server finally runs the optimization algorithm on the output of the Enforcer and the Aggregator and generates the Tunnel Groups.
- The Tunnel Groups are consumed by the TE Database Manager, which generates events for the controllers to consume. These are passed to the SDN gateway and then finally programmed into the network through the controllers.

>[!WARNING] Problem
>We mentioned that some applications are elastic and some are in-elastic, but also, Google allows some contracting applications to be treated as high priority. The TE formulation however, relies on the $\Pi$ for grouping flows, and this function cannot be updated very easily. How can we account for these changes in priority?

### Bandwidth Functions For Applications

Each application is assigned a weight $w_i$. The bandwidth allocated to each app is proportional to these weights. The figure below shows how this assignment can be done.

![[Pasted image 20240221171724.png|500]]

![[Pasted image 20240221172442.png|500]]

These weights are assigned based on the priority of each application.
When two applications are grouped in a flow group with a specific $\Pi$, the total bandwidth (e.g. the demand of the flow group) is calculated by summing their curves piece-wise linearly, and then capping it at a maximum value based on the available capacity.

The optimization output would indicate what is the *Fair Share* ratio that satisfies the optimization problem.

#### An Example

Above, we have provided both a demand and capacity example, and some weight functions. The optimization would have to decide the Tunnel Groups for each Flow Group.

First, we inspect the flow demand graphs. For this example, we have 3 applications. The policy mapping has grouped applications 1 and 2 into flow group 1 and application in its own flow group.
Inspecting the graphs will show:
$$
FG_1(A\rightarrow B) = \begin{cases}
11\times f & 0 \leq f \lt 1.5 \\
15 + f & 1.5 \leq f \lt 5 \\
20 & 5 \leq f \\
\end{cases}
$$
And the demand for $FG_2$ can be assumed to always be $f/2$. 

Now, we must find the possible paths for each flow group and rank them:
- For $FG_1$, possible paths are:
	- $A\rightarrow B$
	- $A\rightarrow C \rightarrow B$
	- $A\rightarrow D \rightarrow C \rightarrow B$
- For $FG_2$, they would be:
	- $A\rightarrow C$
	- $A\rightarrow B \rightarrow C$ and $A\rightarrow D \rightarrow C$

We start greedily allocating for each path until it is filled. The first paths chosen for each flow group are the direct paths, and they are disjoint and thus we fill them until the capacity of one of them is full.

This happens with $FG_1$ first when we have $11 \times f_1 = 10 \Rightarrow f_1=10/11=0.91$. This gives 10 Gbps to group 1 and 0.45 Gbps to group 2.

Now, group 1 will utilize the second path, but group 2 still uses the direct path. The second path for group 1 is $A\rightarrow C \rightarrow B$ which has a bottleneck on the $A$ to $C$ link. Assuming the fair share of this step, $f_2$ is less than $1.5$, we would have:
$$
\begin{cases}
f_2 \lt 1.5 \\
(11\times f_2 - 10) + (\frac{f_2}{2}) \leq 10
\end{cases}
\Rightarrow \text{Underutilized}
$$
So we need to check for the second range of $f_2$:
$$
\begin{cases}
1.5 \leq f_2 \lt 5 \\
(15 + f_2 - 10) + (\frac{f_2}{2}) \leq 10
\end{cases}
\Rightarrow f_2 \leq 3.33
$$
This saturates the $A \rightarrow C$ path, and gives an extra $8.33$ Gbps to group 1 and about $1.66$ Gbps to group 2. Both groups will now move on to the next path. For both groups that can only be through $A$ to $D$ which only has 5 Gbps capacity. Group 1 eats up $1.67$ Gbps and gets satisfied and the rest of it goes to group 2, which will remain unsatisfied.