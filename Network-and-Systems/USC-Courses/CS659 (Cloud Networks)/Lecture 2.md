# View of a Cloud, Intro to datacenters (Cont.)

- A DCN is a basically a building full of racks of servers.
	- Racks have standard sizes, they can house 30 to 40 servers and usually a Top of Rack (ToR) switch that connects them to an up-stream network.
	- The building connects to the internet via a Wide Area Network (WAN).
		- So there is 4 levels of abstraction:
			1.  The Datacenter network connects the racks together
			2.  The Campus network connects the DCs
			3.  The WAN connects the campus to the internet
			4.  The internet itself then needs to route the information across the world to reach the users

## Hosting Applications

![[Pasted image 20240213100545.png|500]]

- In the ingress of the Campus Network or the DCN, there sits a device which we call simply a **Load Balancer**. This box will route the traffic into the DCN in such a way that it satisfies a certain constraint. For example, fair sharing of requests among racks.
- The DCN will house servers, and each server can have hundreds of VMs dedicated to different applications. Usually, each DC would specialize in housing a certain class of application. This is done so that each server within a DC would be **homogenous** in terms of hardware (i.e. any server within the same DC would have the same number of CPU cores or GPU devices). This is important, since VMs might need to be **migrated** across servers for various reasons and would be unwise to migrate something to a place with different resource types.
- A DC may house both the Frontend and the Backend of a web application if required.

### Case Study: GMAIL
- The contents of your mail is indexed and sharded upon multiple servers. When you request for something in your mail, you send a request to a frontend server in one of Google's DCN.
- The frontend has access to the indexing system (which is a Distributed File System). While Google does not clarify  this, it is very unlikely that frontend would wait for a response from outside of it's own DCN, so the interface for the file system would most likely be accessible from the same DCN. So it waits for the DCN to return its query result and then sends the data back to you.
- It is paramount that the servers are very fast, since it is usually the case the each server contains multiple VMs that serve different applications.
  This goes back to Google relying on statistical multiplexing to cut costs.

This pattern of routing/request-responses is not specific to large providers. It is consistent for any creature running on a cloud.

We will discuss almost every part of this process. We first start with a bit of Datacenter design. 

# Jupiter Rising
**Paper:**: [Jupiter Rising: A Decade of Clos Topologies]([Jupiter rising (acm.org)](https://dl.acm.org/doi/pdf/10.1145/2975159))
**More Detailed Paper:** [Jupiter Rising: A Decade of Clos Topologies and Centralized Control in Google’s Datacenter Network](https://storage.googleapis.com/gweb-research2023-media/pubtools/pdf/43837.pdf)

## Background
- Till early 2000s, Cisco/Juniper were the main providers of router devices.
	- ISPs and Campuses would buy very expensive and large router devices. These had:
		- A lot of network devices (NICs)
		- Dedicated CPUs for running network protocols
		- (Maybe) General purpose CPUs for fancy stuff (some people probably are running SQL databases)
		- Up to 1M dollars of cost! (For a single switch!!!)
- **Until Broadcom**
	- They introduced the first modern "Switch" device. Essentially a chip:
		- It was a high performance chip with an SDK that allowed them to interface with NICs and Optical interfaces.
		- They were super cheap compared to Routers.
		- The downside was that you needed to decide on your own hardware (from the motherboard to NICs), and also have the programming chops to actually code with that SDK.
	- We refer to these creatures as ***Merchant Silicon***,

While this is happening, applications being housed on Data centers (at least the primitive version of it in that time) had a huge increase of traffic and user base.

>[!IMPORTANT]
>So the question now is, how would you design a network that is cheap and available, while being reliable enough to handle this massive traffic, using the new switching hardware?

### Google's Original Design (Four Post Dsign)

![[Pasted image 20240213115642.png|500]]

They had:
	- Racks having 40 servers each
	- 512 racks in total, each with a ToR switch
	- 4 Cluster Routers
- Each rack connects to all 4 cluster routers, each with 1 GB link.
	- The upstream capacity of each rack thus is 4 GB.
	- The total traffic that can be supported would be $4\;Gbps \times 512 = 2\;Tbps$. For this Clos topology (we'll discuss this soon), it will also give a Bisection Bandwidth of 2 Terabit per second.

**Problems**: 
1. Each ToR has limited aggregate capacity of 4 Gbps, but each of the 40 servers in a single rack can technically generate 1 Gbps of data. The rack has the potential to generate 40 GB of data.
	Thus, the rack is ***oversubscribed***. For this reason, Google saw much less actual bandwidth compared to the expected 1 Gbps.
2. Applications need to fit under a single ToR, so if you need more than 4 GB, consider your foot out of the door.
3. Still need to write the networking protocols with the SDKs!

>[!NOTE]- A Word on Oversubscription
>The oversubscription ratio is the ratio between the capacity of the NIC of a single server in a rack, and the minimum available bandwidth to that server.
>We hope for a 1 to 1 ratio.
>For Google's original design, each server has 1 Gbps capacity, but with the maximum upstream of 4 Gbps, when all 40 racks shout at the same time, each one gets 0.1 Gbps! So that ratio is 10!.

**Solving oversubscription?**
	- We can get really fast routers. If Google's original design had 10 Gbps routers (i.e. a 512 port router where each port supports 10 Gbps!), we would be fine. We just don't actually have the hardware.
	- Reduce the number of servers in a single rack.
		- That is not good. It would force us to break a large application across racks, and that heavily impacts performance, and may not in fact be feasible for all applications (e.g. some applications are heavily sensitive to latency or changes of it)

**Why care about having 1:1 ratio?**
- A system with 1:1 subscription ratio is said to have **Uniform Bandwidth**. 
	- The main advantage of it is that you can put your application anywhere in the network without performance cost.
	- It also makes them highly available
	- It also provides reliability. In a non-uniform system, if a replica of an application dies and we transfer to a congested portion of the network, we might die again!

>[!FAQ] Definition of Uniform Bandwidth
>If a system provides Uniform Bandwidth:
>- Any server can send at maximum link capacity to any other server
>
>It is very easy to show that having 1 to 1 oversubscription ratio is equivalent to Uniform Bandwidth (just assume the contrary and see that you get a contradiction).

#### Clos Topologies
**Pronunciation Note:** *You don't really spell the final `s` there!*

- Originally created for telephone networks.
- We assume:
	- All switching devices have the same number of ports (called the **Radix** of a switch)
	- All switching devices have the same speed on each port

**Can we make a 1:1 subscription ratio network?**

Yes!

![[Pasted image 20240213130125.png|500]]

We have 2 layers:
	- Aggregation: These switches connect to the hosts with half of their radix, and to the core with the other half.
	- Core: These switches connect to all aggregate switches.

The main defining parameter here is the number of Aggregation layer switches ($N$).
There is no bundled links. A port connects only to another port.

To be non-blocking, the radix of each switch $r$ MUST be greater than or equal to $N$, so $r \geq N$. You would be better off to let $r = N$, but that means that if you want to add something to the system or expand it, **ALL** switches would need to get some upgrade. We generally always assume $r = N$ unless specified otherwise.

As for how routing happens:
- The traffic goes from a host to a ToR switch, from the ToR it goes to an Aggregate, and from there it jumps to a Core switch.
- From there it backtracks to another host.

>[!NOTE] 
>This is a specific form a [[Spine-Leaf]] topology. As such, it is non-blocking.
>Every Clos is a Spine-Leaf, but not every Spine-Leaf is a Clos.

For a little calculation:
- If $r=32$, then:
	- You have 32 aggregates
	- You have $\frac{32}{2}=16$ cores
	- Thus $16 \times 32 \times 40$ servers!

**The main advantage of Clos for building large networks:**
- You can stack Clos topologies **inside** Clos topologies!
- For example, with 8 port switches, a single 2 stage Clos will have $4 \times 8 = 32$ hosts. You can look at it as a switch with 32 ports!
  Stack these boxes in another Clos, and you get exponentially larger topologies!
- None of this would be possible if buying huge amounts of Merchant Silicon wasn't really cheap!

>[!NOTE]- Non-Blocking Topology
>A topology is said to be **Non-Blocking With Respect To Host $H$** if it allows host $H$ to communicate with all other hosts no matter what they are doing.
>A network is *Rearrangeably Non-Blocking* if: 
>- We are non-blocking with respect to all hosts in the network 
>- We are allowed to reroute current traffic from other hosts
>
>This is equivalent to having uniform bandwidth or having 1:1 over-subscription ratio.

### Google's Second Design: Firehose

Firehose was a different design. The goal was to deliver stable, non-blocking 1Gbps bandwidth to about 10K servers. 

Key to creating Firehose was using the idea laid out previously (i.e. creating larger switches from smaller ones). Google however, noting the complexity of actually expanding Clos topologies, adopted a *non-Clos* topology for aggregation layers.
One of the main reasons for this choice, was that the best switches at the time did not have uniform radix, they had a few fast 10 Gbps ports and many more 1 Gbps ports, or just a small set of 10 Gbps ports.
The goal was to utilize the faster ports as much as possible.

![[Pasted image 20240213144046.png|600]]

Firehose consists of 5 stages, but it still keeps the 3 tier topology (i.e. core, aggregate and ToR).

- Stage 1 is the ToR switches, they provide two 10 Gbps uplinks and 24 Gbps downlinks, so each ToR can support 24 servers (although in the paper, only 20 were used for servers, the other 4 were not, we'll see why soon!)
- Stage 2, 3 and 4 are the fabric of the aggregation and core layer:
	- Each switch provides eight 10 Gbps ports, we split them to 2 sets of 4 for uplinks and downlinks.
	- Aggregations are a set of 2 complete bipartite graphs $K_{4,4}$, so **it is NOT a Clos!**
		- Aggregation exposes 32 ports upward and 32 ports downward, each 10 Gbps each.
		- For for utilization, each 32 upward ports should connect to 32 spine blocks and 16 ToR blocks (remember each ToR needs 2 links! This is for failure resilience)
- Stage 5 is the big chunky core layer:
	- Cores are also 32 downward facing 10 Gbps ports, so we need 32 aggregation layers as well.
	- The upper chunky layer consists of 4 switches serving 8 downward 10 Gbps links.

>[!FAQ] What Was The Point of Not Being A Clos?
>One main benefit is that the wiring of the aggregation layer is much less hectic, since aggregation is broken into 2 smaller graphs. The core is still a problem, but there is not much that we can do about that!

So in brief:
- 32 Cores
- 32 Aggregations
- 16 ToR switches
- 20 servers per ToR

So we can serve $20 \times 16 \times 32 = 10240$ servers at 1 Gbps. 

This version of Firehose (which Google refers to as Firehose 1.0) had major problems:
1. The low radix of the ToR switches (2) meant that 2 link failures could very easily disconnect to communicating hosts.
2. Google reported much more server crashes than expected, with long reboot times for each server.

Things were apparently so bad, that Firehose 1.0 **never saw production traffic!**

### Improved Firehose (And 1st Gen. Commercial Network)

Firehose went under a major revision and was reborn as Firehose 1.1.

![[Pasted image 20240213172606.png]]

Perhaps the most important change here is the ToR switches. As you can see, they are **paired** (or *buddied* as Google prefers).
The way this works is that the chips that would operate a single ToR in Firehose 1.0 were installed as a pair onto a single board via PCI. The board would then use a custom made CPU controller to coordinate traffic between these two chips. 

Why do this?
The main benefit here is that since each chip supports 2 uplinks and it is possible to spread traffic over them by utilizing the controller, the ToR as a whole would provide 4 uplinks instead of 2, and this solves the low radix problem of Firehose 1.0. 

The aggregation block was also changed, instead of being 2 $K_{4,4}$ graphs, it is now just one graph to be more resistant to link failures.

Each buddied ToR will now serve up to 2 times 48 hosts with 1 Gbps links (although Google only used 40, and we will see why soon!), with 8 buddied ToR switches, we get $8 \times 2 \times 40 = 640$ hosts under the same aggregation block.

Of course, since the aggregation block is a single graph now which is not complete, we might be oversubscribed. If all hosts shout at the same time, we receive 640 Gbps in the uplink, but each aggregation only provides $32 \times 10 = 320$ Gbps of uplink, so Firehose 1.1 is oversubscribed by 2:1, but it is more resistant to failures. 
It goes without saying that since the aggregation block is just one graph now, it is a huge *nightmare* to actually cable.

FH 1.1 came at the heel of the failure of FH 1.0, so Google understandably had second thoughts on moving it straight to production without a backup, so they were very cautious.
So FH 1.1 was deployed *alongside* the previous 4-post network as a bag on the side:

![[Pasted image 20240213194502.png|500]]

Now we can finally see why they left 4 ports hanging in the 4-post topology and FH 1.1. Note that the 4-post topology only used 1 Gbps links, so we might just make use of the 4 leftover 1 Gbps ports that we could have used for another server!

Google used the legacy network for the anything other than the more specialized intra-cluster production traffic. If something fundamentally broke in FH 1.1 because of a miss-configurations or such, they anticipated it by designing a *Big Red Button* that upon pressing, would re-route traffic completely through the legacy network.

FH 1.1 was relatively bug-free though, but it still had one really annoying problem, **the CABLING!** Apparently it was so bad that Google decided they need another iteration.

This also is true for Clos as well:

**Problem With Clos:**
- Very dense!
	- A nightmare to connect with cables.
	  Each would also need to go a very very far length from aggregate to core!
	- With cupper wire, this becomes really really bad, since there is heavy dispersion at the end of the cable.

Thus, we need **Optical Fibers!**
