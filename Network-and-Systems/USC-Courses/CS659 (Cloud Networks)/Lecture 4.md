Continuing the Jupiter paper. Discussing Watchtower and Saturn.

We were discussing what to do in order to provide external access to the cluster.
There were 4 options:
1. Reserve port links on ToR switches
2. Reserve ports on aggregation blocks
3. Reserve ports on spines
4. Build a separate aggregation block

![[Pasted image 20240214150956.png|500]]

- Option 4 was chosen, i.e.:
	- Any group of ToR can use all external bandwidth.
	- A group of ToR will be used to communicate with the outside.
	- We won't have to touch existing aggregation blocks if we expand

The hierarchy would be Cluster --> DFD --> CFD --> WAN from lower to upper layer. 

>[!FAQ] Why Each Freedome Has 4 Blocks
>The reason is to allow in-place upgrade.
>The operators cannot tolerate losing a whole cluster worth of capacity. If one block was down, the capacity goes down by 25 percent. The operators usually try to keep all DCs below 75 percent load (that is feasible when migrations are allowed, a heavy load DC will migrate applications to a lower load one).
>Thus, having at least 4 blocks makes it somewhat safe to bring blocks down and modify them on a live network.

## Routing Design

The solution Google use is kind of like SDNs (at the time this paper was written, SDNs were just being realized for the first time). The solution they used was called Firepath (later Onix and Orion replaced them).

### Firepath
Each switch had a Firepath client on them. It collected the whole Linkstate information of the router and sent it to a Firepath Master, which reflected it to the other clients.
In contrast to SDNs, here the clients do the actual computation of shortest paths (For BOTH internal AND external prefixes).

The network that carries the information between the clients and the master is a separate Control Plane Network (CPN). So all the link-state information is coming on that network and since its only purpose is that, it does not need to be a very high capacity network.
Having a separate network is necessary, since otherwise, failure in the dataplane will cause loss of control signal, a classic problem of SDNs.

#### ECMP
Since in a DC, there tend to be multiple shortest paths, ECMP is a very attractive way of routing.
To remind you, when in OSPF multiple shortest paths exist, OSPF does not have to make a choice between them. Instead the best solution would be to split the traffic.

>[!IMPORTANT]
>What you DO NOT do is splitting packets of the **same flow** on the paths. This is a rally bad idea, since it can cause packet reordering, and TCP is very slow with dealing with reordering. 
>Most TCPs also assume reordering is very rare, which means that most TCP implementations become really bad under this form of cmulti-pathing.

The way most ECMP implementations work is that they take some hash of the header, and then use that to determine which path we should take. There are multiple choices.

- Hash the whole header, not really good, since header data can change depending on the protocol and can still cause reordering when it sends packets of the same flow on different paths.
- Hash only the IP header. Not efficient, since it bundles all the traffic to a destination from a source on a single link.
- Hash IP and TCP header. Much better. Since the TCP ports tend to be somewhat random (the OS usually picks one), it means that each TCP flow will be assigned to a path at random, but never will that path change so long the flow is alive.

#### Neighbor Discovery
There are 2 main goals of ND:
1. Get neighbor ID on each port and generate link state
2. Check against the neighbor you **expect** to have (the operator usually pre-defines neighbor to neighbor relationships)

This is key for Firepath, since the relation between clients and the master is key to the performance of the system. We should note that this is only possible because of the networks *regularity*, the DCN is very orderly and each switch has a fixed, predefined role. Such a solution may not work in a WAN topology for example.

See slide 51 to see what other choices were possible.

It should be stressed that ECMP is the most important tool here. Without it, achieving uniform bandwidth would be impossible.

# Jupiter Evolving:

**Paper:** [“Jupiter evolving: transforming google’s datacenter network via optical circuit switches and software-defined networking”](https://doi.org/10.1145/3544216.3544265)

## Background: 

Before moving on, we need some bakcground about what happened with the networking infrastructure since the last paper.

### Waveform Division Modulation (WDM)
The main technology that powers fiber communication is WDM. 
The details need not be discussed, but it is important to note that:
- Multiple streams of data may exist on the same fiber
- The rate of each stream can be tuned (although the fiber as a whole has a fixed capacity of course) but allocating wavelengths.
- Switching between fiber links can be done with Optical Circuit Switchers (OCS). It is essentially a patch panel (i.e. a board that creates the circuits for switching, although unlike a telephone patch panel no human operator is needed)

![[Pasted image 20240214084227.png]]

The OCS is reconfigurable, but the number of ports on it are somewhat limited.

>[!IMPORTANT] The Main Advantage
>The same fiber can hold multiple streams with **tunable** bandwidth. This is in direct contrast to Ethernet, where only an upper bound was made and the actual bandwidth depended on the network condition and traffic.

### The Jupiter Problems

1. The network, while working nicely, was still a pain to *evolve*. You do not want to (nor perhaps are able to) throw away previous network hardware. 
2. Capacity requirements have sky-rocketted thanks to ML and GPUs. We have the hardware to achieve this capacity, the problem is installing it!

Clos networks just do not support evolving very well, since it is needed for one to prepare all the Spine bocks. This is:
1. Very expensive!
2. Not particularly fine-grained network speed increase.

To make matters worse, if you wanted to upgrade aggregation block because Broadcom roled out a shiny new chip, you HAVE to upgrade the Spine first! Because the bandwith would be the minimum of the port bandwidth of the Spine and Aggregation blocks.

Google's solution was to replace almost all links with fibers, and use OCS to increase network capacity at will.
One problem is that Google builds their own OCS (it is as every bit mind-boggling as you may think, like WTH!). In fact, they had even more radical of an idea. They replaced the **entire** spine with a gigantic OCS layer.

![[Pasted image 20240214092033.png]]

This provides a new architecture, where we can go from any input to any output. This makes cabling easy and also means that the spine would no longer limit our ability to expand. In its current form, you can see an implementation of it in the figure above:
- The initial two ports can be directly connected.
- Once a new port is added, we can connect that to the previous 3 ports with just the spine itself.

This means that we can just connect things to the OCS and let them communicate with anything else on the network. 

![[Pasted image 20240214093119.png]]

Above is a meow detailed view, the above line is the actual topology with the OCS and below is the *logical* view of each topology, any aggregation block connected to the OCS will live on a bipartite graph.

**The main consequence of this design is that aggregation blocks are directly connected. This means that we no longer have uniform bandwidth as direct connection would imply that the bandwidth of other ports needs to be lowered**. This design is called **Direct Connect**.

On this uniform topology, if demands were also uniform, then this would be perfect, but since demand is not uniform and ECMP isn't viable here, we cannot rely on our previous methods for deployment, thus, Traffic Engineering (TE) is required.
TE is complimented with Topology Engineering, which allows one to dynamically change the bandwidth of the links.
### Direct Connect Details

The OCS provides the abstraction of being a single gigantic device, but of course, it is made of smaller boxes. Each box is called a Data Center Network Interface (DCNI). Aggregation blocks connect to these DCNIs and the OCS will connect their uplinks to each other as needed. 

As for how this is actually managed, lets walk through an example:

![[Pasted image 20240214101332.png]]

>[!FAQ] What's With The Colors?
>Google uses a 4-colored design, where a color signifies a **Failure Domain**, a series of devices that can expectedly go down together in the event of a catastrophic failure (e.g. large scale power cutoff).
>Google's heuristic is to have 4 colors, since the DC operates around 75 percent capacity by default, thus expecting a single failure would mean that the DC could still function acceptably.
>

Here, we start with an initial uniform traffic matrix, where A and B both request  50 Tbps (more specifically 51.2, coming from $512 \times 100$ Gbps). The DCNI connects the two uniformly via the OCS.

When block C is added, also rocking 512 uplinks, it is directly connected with the others. This cuts the previous demand in half for everyone and thus each pair can only communicate with 25 Tbps of traffic. This is the **Topology Engineering** step, and it makes sure that a uniform demand can be met over the topology in this particular case.

In general, any split of traffic can be done with Topology Engineering, the only thing is that since it happens over a longer period of time, it can only reflect the ***mean*** traffic matrix, we still need to cope with **variations** of the traffic matrix around the mean.

So how would the variation look like?
If A wishes to communicate with C at a higher rate, with this configuration, the direct link will be a bottleneck, thus A needs to use the auxiliary 2-hop path through B to achieve this.

Here is how we would implement that:
- We implement WCMP routing at the egress of A, we want 25 Tbps to go through $A \rightarrow C$ path and 5 Tbps to go through $A \rightarrow B \rightarrow C$ path, which means that we give a weight of 5 and 1 to them.
- Thus we get an aggregate of 30 Tbps on the ingress of C , but B would have to live with only 20 Tbps.

This is how **Traffic Engineering** helps the network adapt to the variations in the traffic matrix. So in summary:
- **Topology Engineering to operate on the mean traffic matrix**
- **Traffic Engineering to operate on variations on the traffic matrix**

The second line (numbers 4 to 6) show something even more fancy. When blocks C and D were added we can also still do the Traffic/Topology Engineering to match the demand, but if the blocks C and D were to receive an upgraded ToR that lets them communicate at a higher speed, then the OCS can be expanded to provide more bandwidth **Just For Them!** 

Equivalently, it means that each row in the traffic matrix **Need Not Necessarily Sum To The Same Number!** (you can see this between figures 5 and 6, in figure 5, rows 3 and 4 sum to 50, but in figure 6, they sum to 62!!)

