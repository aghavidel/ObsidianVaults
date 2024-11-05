**Title:** Achieving High Utilization with Software Driven WAN
**Link:** [Achieving high utilization with software-driven WAN (acm.org)]([Achieving-High-Utilization-with-Software-Driven-WAN.pdf](https://www.microsoft.com/en-us/research/wp-content/uploads/2013/08/Achieving-High-Utilization-with-Software-Driven-WAN.pdf?msockid=3c8c97b7b46b65e6396b8334b54664d4))
**Conference:** SIGCOMM 2013

# Abstract
The paper presents SWAN, a system designed to push updates into inter-data center networks based on the required demand. The main point of optimization here, is that if the updates are not coordinated carefully (i.e. if we just let things go by default, switches will install them at different time instances ...), we will cause significant congestion.
As such, we must be a bit more careful about what we do. 
The paper combines SDN and established WAN management techniques to achieve a high throughput solution for managing such a network.

# Intro.
The main concern of WANs in the current day and age, is connecting data centers. These dedicated networks carry GBs or TBs of traffic, have unique traffic characteristics and are extremely expensive resources.
However, current (i.e. 2013) datacenter showed poor resource utilization.
This has many factors, but to name a few:
- There is no coordination between resource demands. Services send traffic whenever they want and as much as they want. As such, the network is provisioned for the highest peak of traffic, which means that is on average, underutilized.
- Current resource allocation techniques are distributed and greedy, which can mean that it is possible for them to converge to sub-optimal allocations because they do not have a global view of the network.
  One example is [[MPLS-TE]] that uses [[MPLS]] for resource provisioning, but can also cause inflated latency in networks.

Combine the above, and you get utilizations in the ball park of 40 to 60 percent. Not ideal.
The Software-defined WAN (SWAN) aims to solve this problem by using a logically centralized view of the entire network. In order to provide good service however, we should be careful about how we update the data plane.
This is because when an update is to be pushed into the network, congestion may happen during the time where the network is in transition between it's previous and current states. This can be especially problematic when the post-update traffic arrives *before* the pre-update traffic has been fully processed.
The main problem is that updates to switch states are not atomic, and this effect only worsens with busier networks and higher RTTs. Now there have been attempts to make these updates atomic (see [[Abstractions for Network Update]]) but they are not well suited to a scenario involving many flows and larger networks.

The key points that allow for a congestion-free update to switches is:
- You cannot update a network that is in full capacity without congestion. As such, you must reserve a small *scratch bandwidth* of say  $s$ on every link (like 10 percent of each node) that allows for congestion free updates. 
  We shall prove that such an update can be done in $\lceil \frac{1}{s} \rceil - 1$ steps.
- Decrease the total number of rules required to push into switches, by using the minimum amount of paths  required to carry traffic.
  This in turn, also makes sure that capacity utilization is held at maximum.

We discussed SWAN in the context of [[USC-Courses/CS659 (Cloud Networks)/Lecture 14|Lecture 14]] for CS656, but here, we try to focus more on the algorithmic part of it, since we will refer to it for TE solutions later on.
As such, we will skip most of the motivation aspect and background and go directly to problem formulation.
