# Direct Connect

Recap from previous lecture.
We removed the spine layer completely and replaced it with the OCS layer. This gave the **Direct Connect** scheme, which gives a direct path from one aggregation block to another.

The downside was that it removed the uniform bandwidth since we had no ECMP. We need a form of routing that resolves that. We use a scheme called **Demand Oblivious Routing**, in contrast to **Demand Aware Routing**.

We have seen ECMP, which is one of the simplest versions of oblivious routing. These routing schemes quite simply do not care what the traffic is.

Direct Connect is feasible since:
- It handles predictable, non-worst case traffics well
- Allows for traffic engineering and topology engineering solutions

It is also desirable, since:
- It reduces capital cost (CAPEX, upfront cost of infrastructure) and operational cost (OPEX).
- It is extensible.

## Example

![[Pasted image 20240214101332.png]]

The example is two aggregation blocks, A and B, which use 512 uplinks to the OCS. The OCS connects very uplink of A to an uplink of B. We assume a uniform traffic matrix between them, so 50 terabits per second from A to B and B to A.

Google uses the **4 color** scheme, where of the OCS cores, each quarter of them is on a separate power domain. This means that we can assume at most one quarter will go down in the event of a single power outage, so we have at least 75 percent of capacity.

Each failure domain has its own power infrastructure and also an SDN controller that configures them. These power domains are controller and managed completely independently, and that is an important requirement for safety, and one that Google was very careful with.

### Fabric Expansion
We add a new aggregate block C. It too has 512 uplinks, so to connect A to B and to C, we will put 256 links between each of them. We now allow a uniform traffic matrix of 25 terabits between all of them.

Now assume the demand changes. We now want 30 units of traffic from C to A and back, and 20 for others.
This is still feasible, but how do we achieve it?
One way is Traffic Engineering, route 25 units from A to C directly and 5 units from A to B and then to C (so a 2 hop path).

### Partial Expansion
We now add an aggregation block D, which only has 256 uplinks instead of 512. 
What to do? Symmetry lets us infer that the 256 link should be split evenly to all 3 other blocks (so 2 of them get 85 and one gets 86). If the traffic matrix also exhibits the symmetry (i.e. the traffic from D to all other 3 is the same, and the traffic among A and B and C is also the same), and so we will have 213/214 links between the other 3.

This is very basic topology engineering (although it needs to be able to be done programmatically).

### Block Refresh

Assume blocks C and D have 512 links at 200 Gbps. If the previous network used 100 G and we were using a spine instead of an OCS, then we would still be limited to 100 Gbps!

>[!NOTE] About  Aggregation Blocks
>The paper uses different colors to show different failure/power domains. Each aggregation block (also called Superblocks) contains 4 separate Middle Blocks (MB) (each different colors)
>Each MB is a blocking design, and blocks in different power domains are completely isolated, only the OCS provides connection between them.
>
>Google goes to great lengths to make sure power domains are isolated, and that is the main reason that Jupiter is a blocking topology. 

### Rack-to-Rack Traffic
- If two racks are connected to the same MB, their traffic never touches the OCS
- Racks in different MBs:
	- If there is a direct path, then the traffic bounces off the OCS
- Racks in different MBs and a 2-hop path.
	1. Traffic bounces of OCS
	2. Then the layer MB in the superblock
	3. Then to the other rack

## Traffic Engineering

The goal of traffic engineering is to satisfy block to block demand while:
- Traffic matrix changes with time
- Traffic matrix could be unpredictable
Without changing the topology.

The key ideas are:
1. Use indirct paths via one other aggregation block
2. Change topology when no feasible solution exists

We usually use a traffic matrix. These matrcies need to be estimated, and while their main specifiers (mean traffic for example) is quite predicable, there is uncertainty around the exact value always (there is large variance sometimes).

We use TE to cope with these uncertainties, noting that topolgoy engineering takes some time, traffic engineering is the only feasible way to handle them.

### Hedging

Since the traffic has uncertainty, one way of solving it would be to provision with an extra **headroom** equal to some upper bound on that variance. This is too much usually, so a better way is hedging.

Hedging would split the traffic across all available paths, and only provision for the the maximum amount of uncertainty among the traffic, devided by the number of paths.

For example:

![[Pasted image 20240214110740.png]]

Here, the path from 1 to 2 at this moment, requires a large burst, which increases utilization by $\frac{\delta}{C}$. We could provision this extra headroom on all links, but that would be too much. We can make use of the other 2 paths, and instead provision $\frac{\delta}{3.C}$ on them instead.

The reason why we can provision just for the max is because these uncertainties can be assumed to be uncorrelated, we won't see them happen exactly at the same time. We refrain from doing this for longer paths, since it worsens congestion (it can cause packet re-ordering and TCP really hates that!).

### TE Algorithm More Formally

We collect $D(i, j)$, some estimate of the traffic required to be handled from block $i$ to block $j$. We also receive $C(i, j)$, the available capacity between the two blocks from the operator.

A TE algorithm will receive these as input, and spit out a list of $\langle p_k, f_k \rangle$, where $p_k$ is some single-hop or indirect path from $i$ to $j$ and $f_k$ is the fraction of the demand that we should have on it (think of it as WCMP weights).

The objective here is to minimize the maximum link utilization. This is a multi-commodity flow problem that can be solved pretty efficiently. Usually, these systems employ **Max-Min Fairness** solvers, which mean that it will try to pick a solution that minimizes the maximum link utilization. Why is this good? It provides a certain degree of fairness, but we won't dwell on that for now.

We deviate from the usual norm though, **if there are multiple solutions, we will split the traffic on them via hedging**. So the whole problem is solved with a constrain that forces the solver to spread at least a pre-defined fraction of the traffic on the solutions. This spreading factor must be estimated for each cluster, and it reflects a large amount of the uncertainty in the network.

>[!NOTE]
>We don't take hedging for granted. It still causes increased latency. But in the face of large uncertainty, hedging is a really good solution.

A Topology Engineering algorithm would do the same, but it would also spit out link capacities as $C(i, j)$. This is much more complicated, so it is nice that for now at least, we need to only do this a few weeks at a time.
## Control Plane

Jupiter uses Orion, Google's own SDN controller. Each failure domain has a separate Orion controller as its master. Each 4 Orion controller is completely unaware of the other, and then, Google uses high level applications to tell each one what to do.

We will study Orion later.

### Handling Failures

When a failure domain goes down, the network behavior depends on what actually went down. Here, we consider Control Plane failure. Google defines 2 types of failures:
- **Fail Static (or Fail Open):**: The network continues with its most recent state, unable to be programmed, but still functioning.
- **Fail Close:** Complete halt of operation. Like power outage.

The controller itself also lives in a completely separate network to make the design easier and safer.