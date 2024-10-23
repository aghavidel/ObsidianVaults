**Paper:** [FBOSS: Building Switch Software at Scale](https://dl.acm.org/doi/pdf/10.1145/3230543.3230546)

# Intro.

This paper comes from the folks at Facebook, and concerns the implementation of their own switch software stack (i.e. their own Network Operating System) for all of their datacenter switches. 
We had previously seen FBOSS in the CS 659 lecture where we discussed Facebook's BGP usage for their datacenters. See [[USC-Courses/CS659 (Cloud Networks)/Lecture 7|Lecture 7]] for a refresher!

But first, why even bother doing this? Vendors like Cisco and Juniper already provide you with a switch operating system, what's wrong with that?

The concerns here are:
- *Deploying* at scale
- *Managing* things at that scale

As we learned from Jeff Dean's [The Tail at Scale](https://cacm.acm.org/research/the-tail-at-scale/) paper, even improbable hiccups can cause a problem when we reach a large enough of a scale. A more optimistic version of that principle would be that even small improvements would help immensely when you go into a large enough scale.

The thing is that these conventional Switch OSes come with a huge array of drivers. Most important among them are those for managing proprietary ASICs that are at the heart of these switches. The curious thing is that as long as these most fundamental hardware components are kept consistent among products, all of the vendor products can still make use of the *same* massive set of drivers and software that they made for other devices and only need to make small tweaks.

The problem is that this really only works on the assumption that customer usage are also correlated among all of these devices, whereas that quite simply is not the case. This is really good for the vendor, since they save a lot of money and generate much more by just handing out the same software to the users, but it becomes a headache for the people who buy this sort of devices in large quantities to make data centers! Google learned this listen a while ago and Facebook is following the same thing essentially, except that Google started with the hardware directly, while Facebook was more concerned about the software (which makes sense, since when Facebook was thinking about this stuff, Google had already announced that they were using Merchant Silicon for their whole fleet and everyone realized the potential).

So here is what we know:
- When you build a DC, you really only need layer 3 networking, and that is a pretty small subset of the NOS features that switches come with. Microsoft's [SONiC](https://azure.microsoft.com/en-us/blog/sonic-the-networking-switch-software-that-powers-the-microsoft-global-cloud/) for example (another NOS which became popular when Cumulus Linux was acquired by Nvidia and dropped support for Broadcom) is designed to be very modular, so that people can develop a very lean software stack.
- If you leave the software stack to the vendors, you cannot *innovate* on it easily, unless there is a high costumer demand for it on the vendor side. Facebook cites IPv6 forwarding, which while it was implemented from their vendors quickly, some of their features took really long to roll out, so they did it themselves!

All of these motivated Facebook to implement their own NOS, which finally became FBOSS.

# Design Principles

FBOSS cites two main design principles:
1. **Switch-as-a-Server:** Treat the switch software the same you do any other server software
2. **Deploy-Early-then-Iterate:** Deploy the basic service very fast, and then iteratively add to it to minimize total software complexity.

## Switch-as-a-Server

The insight from Facebook here is that deploying software needs to be quick and fast, rather than fully feature-free or bug-free. This is what allowed them to scale their software services heavily, they deployed the minimal set of features needed and then refined and modified it as they went.

This is also where they preferred to stay away from proprietary systems. For example, for their databases, they just took an Open Source implementation and heavily modified it to suit their own needs, which provided to them the most basic requirements that they needed without any extra fat.

Having less complexity in their servers was a big deal, and so they could not see a good reason why applying that to the switch software wouldn't be beneficial. Some refinements to this approach are needed though which we shall discuss soon ...

## Deploy-Early-and-Iterate

Facebook mentions that they did NOT deploy with full-features!
This of course has to do a lot with their DC design which only needed BGP (a subset of it also), so they were not worried about leaving things on the table.

This made the software much less complex and also made it much easier to fix problems as they came. Of course, some times it would also mean that Facebook could have underestimated the required feature set.

>[!EXAMPLE]
>Facebook mentions that they did not deploy Control-Plane Policing (CoPP) which made their control plane vulnerable to overloads, and in turn caused some of their BGP sessions to timeout.
>So they quickly implemented QoS support for control plane packets and fixed it!
>
>On the other hand, they did not (and have not!) implemented any Spanning Tree Protocol for their system! Which is understandable when you only have layer 3 networks!

# Hardware and Software

Facebook customized their own needs, though nowhere to the extend that Google did for Jupiter for example. They did make the conscious choice of using more powerful CPUs for their switches, since again, they are treating these switches as servers, and switches have notoriously underpowered CPUs compared to actual commodity servers.

They also had the freedom of using general x86 CPUs which allowed them to install Linux and its derivations on the switch to provide basic OS functionalities. The CPU, while cheap, is still over-provisioned to anticipate extra load.

![[Pasted image 20240308191003.png|500]]

FBOSS interacts directly with device drivers to make sure that configuration is done with minimum friction. This is easy to do with simple C++ code very much because FBOSS has Linux to support it as a kernel. The code is Open Source and very well maintained, with active contributions from other users (although it is hard to say how much that contribution actually affects Facebook :))

The general architecture of the switch is as follows:

![[Pasted image 20240308191831.png|500]]

So basically:
- FBOSS and its dependencies come up over Linux and then use the switch SDKs to interact with the switch ASIC. As for what it will do with it:
	- It has access to the TCAM, and so it can configure routing
	- It has access to all other buffers and memories, including SRAMs, so it can program ACLs and whatnot.
	- It also receives updates on every piece of hardware attached to the ASIC, so if a port status changes, the ASIC will send an event to FBOSS via the SDK
- OpenBMC is an extension over Linux (more specifically, it is a Linux distribution itself) that allows easier programming of hardware and interaction with drivers for specific things. It is especially useful for ToR switches, servers and RAID appliances.

It makes the picture quite clear to think of FBOSS as part of an Operating System designed specifically for switches. That picture is much more clear if you look into FBOSS itself:

![[Pasted image 20240308231028.png|500]]

It is the *exact* same picture for a general monolithic OS:
- You have the hardware itself, which is mostly abstracted by the SDK, which the vendor provides and QSFP services which are like interrupts 
- There is *Hardware Abstraction Layer* which is implemented per vendor (so for example Broadcom would have its own code, and Mellanox would as well) and gives a unified interface for the upper layer
- There is the *Hardware Switch* `HwSwitch` provides the true generic hardware abstraction (e.g. it provides access to an abstract interface object that reflects the true QSFP interface in the hardware, but supports API calls like UP or DOWN)
- And then there is the Software Switch `SwSwitch`, the full abstraction of the Datapath that can be used for state management and configuration.

All of this is deployed *per-switch*.

>[!FAQ] Where is the Control Plane?
>The control plane isn't part of FBOSS. FBOSS does interact with *Robotron*, which is Facebook's network configuration manager, but the actual routing configuration is handled by a centralized BGP daemon that isn't part of this architecture, it is just sitting somewhere else in the DC.

>[!IMPORTANT] Why is the QSFP service completely separate?
>The QSFP service is supposed to manage the QSFP ports, and monitor their state. The service, according to Facebook, continuously evolved and could actually fail at times, all of which required a restart of the FBOSS agent as a whole. 
>
>The decision to move the QSFP service to a separate process was a conscious one to make sure FBOSS as a whole wouldn't need be repeatedly restarted. This of course makes things more complex though, since now the two processes need to be synchronized to keep things consistent.

# Lessons From Deployment 

The majority of bugs and outages in FB's data centers are because of software failures, including FBOSS:

![[Pasted image 20240308233806.png|500]]

FBOSS is developed continuously and changed by a small amount each time. Typical deployment tests of course still apply to it, like checking compile warnings or memory leaks, but most of these errors happen only if it is deployed *network-wide* and that means that new deployments must be constantly monitored, at least in the beginning.

To this end, FB uses their own deployment sentinel, `fbossdeploy`, which upon an update:
- Checks for abnormalities, most importantly it checks for *delays in BGP convergence*
- Network attributes like link failures and reachability information

The software deployment has 3 stages:
- *Continuous Canary*, where the code is pushed only to a few switches automatically. Switches of different types (e.g. different vendors and whatnot) are considered equally. If a failure is detected, the software will quickly revert to the previous version without a hiccup.
- *Daily Canary*, same as above, but with more switches and for a whole day. This catches slowly emerging bugs like memory leaks.
- *Staged Deployment*, this is where an actual human gets involved! This step pushes the code to all production level switches. This is done to all production switches and if during this, more than around 0.5 percent of the switches experience failure, then the process is halted and engineers will investigate it directly.

## Experiences 
