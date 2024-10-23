# Cloud Computing Intro.

We first start by discussing the old days. 
In the 90s, enterprises would use *clusters* of machines:
- These were multiple servers connected in a LAN
- They reduced labor costs, as they took over doing things like billing, keeping personnel records and accounting
- A unit of these was held in a **rack**, a series of servers under a single power management unit, usually connected to a L2 switch on the top referred to as the **Top of Rack (ToR) Switch**

![[Pasted image 20240213084151.png]]

To scale horizontally, you would add more and more racks and buy more switches, keeping them connected in a private network as long as needed.
Now this looks fine, but there are problems:
- There is a particularly high up-front cost, you *HAVE* to but a single, very expensive ToR switch even if you got only a few servers. These up-front costs are referred to as **Capital Expenditures** or Capex
- They are *inelastic*, unless you guarantee a constant heavy load all day (which you can't) a good portion of your resources will be **under-utilized**, since you would have to provision for the worse case scenario, whereas that would not happen often!
- There is also high operation expenditures (Opex):
	- Significant cooling infrastructure is needed, you can see it in the picture above even (see those vents on the ground?)
	- You bet these things are power hungry! 
	- You need security as well, if you are going to put records into something, you should be sure someone is keeping an eye on the physical rack itself.

It was with the recognition of this limitation (mostly the recognition of that fact that they would remain underutilized) that some crafty individuals noticed that they can take over this infrastructure on behalf of other people!

Fast forward a few years, and the notion of a **Cloud** was born.

>[!FAQ] Definition of A Cloud
>There is no textbook definition of a cloud, but it is usually referred to a system with the following properties:
>- It is a **distributed** computation and storage system
>- It **dynamically** scales either to the required load or costumer requirements
>- It is **abstract**, meaning that the costumer itself need not know what sort of complex operations need to be handled to provide the above 2
>
>Those who provide these systems on-demand are referred to as **Cloud Providers**.

Letting a dedicated party take over these systems would mean that:
- They can be much more **elastic**, the provider would buy infrastructure on-mass and a bulk purchase is much cheaper than multiple smaller ones, and this means that they can scale much more than a normal enterprise
- They benefit from **Statistical Multiplexing**, what this means, is that:
	- Assume we have many (say $N$) costumers with mean service requirement of $m$ and peak requirement of $M$. For sake of simplicity, assume that the the peak occurs with a probability of $0 \lt p \lt 1$  for all costumers.
	- To provision for the worst case, would require a total of $M$ resource units. A single costumer would have to be ready for such an event with probability $p$.
	- The cloud provider however, would observe this event only with probability $p^N$, if we assume costumers utilize the resources independently (which is not true, but not too off either); thus, the provider won't have to allocate $M.N$ units to make sure everyone is happy!, what they should allocate would perhaps be some amount higher than $m.N$ which depends on the utilization distribution.
	- This means that the cloud provider can not only get away with not assuming the worst case, they can make sure that they can maximize utilization, since the probability of at least one of the costumers being in maximum utility is $1 - (1 - p)^N$ which only increases with the number of costumers!
- They can provide much higher wide area network traffic capacity, since they have many more ToR switches.

This system has only grown larger overtime, with Google coming out on top for various reasons. Today, they have a mind bogglingly massive network that spans the whole planet:

![[Pasted image 20240213092010.png]]

Each site is connected via trans-continental fiber optics links (i.e. links literarily on the ocean floor!).

## The Hierarchy Of A Cloud

This system illustrates the **Wide Area Network** (WAN) that connects each **Point of Presence** (PoP). Each dot is a privately owned site by Google. Logically, it looks like the following:

![[Pasted image 20240213095649.png|500]]

So the WAN is the top of the line that connects the largest units in a cloud. The WAN relies on BGP and IGPs for routing with Traffic Engineering for optimization (we'll come back to this soon).

Each PoP (or site) is also referred to as a **Campus** in literature, which consists of a private network connecting several buildings (the building set is the actual campus).

![[Pasted image 20240213095948.png|500]]

Each building would house at least one cluster. Today, these clusters are referred to as **Data Centers (DCs)**, although at times it may refer to the campus itself (which may or may not be accurate depending on the size of the campus).

A cluster would have its own network, which would be a large number of racks setup in rows of servers, connected via a dedicated **Data Center Network**. The network connects the ToR switches and might require on-site controllers, since failures can occur within a DC for various reasons.

![[Pasted image 20240213100316.png|500]]


