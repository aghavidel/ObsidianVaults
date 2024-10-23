- **Full Title**: *Orion: Google’s Software-Defined Networking Control Plane*
- **Authors**: Andrew D. Ferguson, Steve Gribble, Chi-Yao Hong, Charles Killian, Waqar Mohsin Henrik Muehe, Joon Ong, Leon Poutievski, Arjun Singh, Lorenzo Vicisano, Richard Alimi Shawn Shuoshuo Chen, Mike Conley, Subhasree Mandal, Karthik Nagaraj, Kondapa Naidu Bollineni Amr Sabaa, Shidong Zhang, Min Zhu, Amin Vahdat
- **Where**: NSDI
- **When**: 2021
- [Link](https://www.usenix.org/system/files/nsdi21-ferguson.pdf)

## Useful References 

16, 17, 12, 13, 11, 22
Jupiter: 28
B4: 15

## Abstract

Orion, is Google's SDN controller, now operating on the entirety of their datacenter ([[Jupiter]]) and their WAN ([[B4]]). Orion's main concern is to be scalable, and amenable to global control hierarchies, allowing it to be used in different networks with ease.

Orion relies on two main design principles:
- Microservice based architecture for scalability
- A Pub-Sub central database that allows for tightly couples distributed controller
Emphasis on the fact that Orion is a *distributed* controller rather than a purely centralized controller.

While SDN promises many things and practical implementations have been very successful, the main problem is that using it in an enterprise network requires a "production-grade control plane that meets or even exceeds current networking performance and availability levels", which is yet to be achieved.
Also equally important, is the fact that if the SDN is to be useful in real world scenarios, it must be able to operate along it's legacy peers, something that is usually handled with ad-hoc solutions which may or may not be feasible for enterprise production.

Here, the main building blocks of Orion will be discussed, and what actually makes it different from other implementations. Though bare in mind, the main contribution of Orion lies mostly with it's control plane design rather than it's implementation, which means that the ideas presented here can be re-implemented in other projects without loss of generality; With that said, neither Orion nor SDN are solutions to all difficulties, only the key challenges in designing Orion should be taken into account for future work, the rest concerns Orion and Google's network only. 

One of the main building blocks of Orion is the Networking Information Base (NIB, probably following the conventions of FIB and RIB). This is probably the main component of Orion that allows for the rapid and independent development of features on Orion.

>[!IMPORTANT] 
>The NIB sequences and replicates updates through a key-value abstraction.

### Key Challenges Of Orion

1. Logically centralized architecture requires very high process power and slick memory management to coordinate SDN applications (which are very loosely coupled most of the time).
2. It is important to differentiate what's happening to the control plane from what's happening to the data plane, as the two are now largely decoupled. For instance, failures detected in the control plane are not necessarily failures in the data plane itself, it may just be that the connection between the data plane and the control plane has been disrupted.
3. The classic balance of centralized vs de-centralized in SDN controllers is still a problem here as well. In brief:
    - Centralized design is easier to manage and implement, but scales poorly and is susceptible to failure.
    - De-centralized design can be robust to failure and can scale very well, but can be exceedingly hard to manage or implement.
4. Integrating non-SDN enabled networks into the ecosystem usually requires using existing routing protocols (mainly BGP) which are usually a poor match for SDN based approaches, requiring a lot of ad-hoc solutions to make them work.

## Design Principles of Orion

The following section will describe the main design principles of Orion. These were divided into 3 main categories by the authors, including *environmental*, *implementation* and *architectural*.

- **Environmental** principles, relate to the type of network used for Orion, and mostly concerns practices used within Google's network, which are nonetheless, relevant to it's successful deployment.
- **Implementation** principles relate to software design and choice of patterns.
- **Architectural** principles relate to SDN usage within Orion and the main function of the control plane (this will be our focus if we intend to go beyond just Orion).

### Environmental Concerns

Since Orion will be used in Google's own network, the following need to be considered for it's production grade usage:

#### Intent Based Networking

This is a much more general principle, and goes beyond just Orion's usage. Many enterprise networks these days follow this principle, as it makes it much easier to manage and configure a large network at the same time.

Not much explanation is probably needed here, but Intent Based Networking (IBN) is the principle that:
- Network configuration is concerned solely with specifying the intended state of the network, rather than the precise sequence of actions that lead to that state, given the current state of the network.
- Network operation on the other hand, simply acts on the output of the configuration output with a series of pre-defined or calculated steps to transform the network state to the intended state.

In layman terms, configuration only describes "what" we should do, and operation only describes "how" it should be done. IBN paves the way for designing more scalable and robust networks, since:
- Separating configuration and operation means that configuration can be described abstractly, allowing for easier management.
- A large part of the network operation can be pre-computed, or pre-defined, meaning that the network can scale much better to large scale configuration changes.

#### Control Plane and Physical Failure Alignment

In layman terms, this means that failure in the control plane of the network should not lead to the same thing in the data plane. This means that using fully centralized controllers is not an option, as any failure in the control plane will instantly be seen in the data plane.

### Architectural Concerns

Here, we get into one of the main things to take away from Orion.

#### Failure Modes

One of the main concerns with SDNs is that, if a node in a topology fails (which even detecting that is already a problem of it's own), how should the controller react to it?

In a perfect world, the controller may compute the intended state over a topology that simply does not include the failed node, however, two problems can usually occur:
- Most failures usually arise from congestion, and reconfiguring the entire network can actually make congestion *worse*, which in turn means that trying create the topology without the failed node, will probably put another node on the brink of collapse.
- Actually calculating the entire state of the network is usually not feasible, either in terms of resources, or just time.

To discuss how Orion tries to work with this issue, we need to first describe how Orion monitors network functionality. Orion associates a ternary state of health to each node under it's provision. Nodes may be either `HEALTHY`, `UNHEALTHY` or `UNKNOWN`. 

- A node is `HEALTHY` if it recently participated in a control communication with the controller.
- A node is `UNHEALTHY` if either it's neighbors are `UNHEALTHY`, or indirect signals from the switch imply that it is not working as intended.
- A node is `UNKNOWN` if it has yet to participate in a communication with the controller, or if it suddenly stopped doing it for no apparent reason. This *may* be a malfunction, or (more probable, especially in larger scales) the node has simply been unable to communicate with the controller, despite having a perfectly functional data plane.

Orion aggregates these health states over the entire network and combines them, to decide how it should react to switch failures. Orion operates on either `CLOSED` or `STATIC` failure modes.

The main principle is that:
- When failure is wide and correlated (meaning that failure of a node seems feed into the failure of another node), the controller enters `STATIC` mode, where it does not remove failed nodes from the topology and continues to operate on the same topology with minimal change to networking operation, until either the state changes or a network operator intervenes.
- When failure is localized or uncorrelated between a small set of nodes, the controller enters `CLOSED` failure handling, and tries to conservatively reroute traffic around failed nodes.

![[Pasted image 20230109195944.png]]

>[!TlDr]
> In our experience, occurrences of Fail Static are fairly common and almost always appropriate, in the sense that they are not associated with loss of data plane performance. They are most often associated with software failure in the controller or loss of connectivity between the switch and the controller.


#### Control Plane Networks (CPN)

One of the key considerations in SDN, is the fact that the controller itself needs to maintain connection to the network under it's provision, through *some network*, which we call the Control Plane Network (or CPN).

There are two ways to handle the architecture of this network:

- The CPN can either be part of the data plane itself.
- The CPN can be an out-of-band network, reserved especially for the connection between the data plane and the control plane.

Most cases rely on the second choice, and ideally, this is done with a network that maintains very high availability, but in practice, this opens up the network to another possible state of failure, since if *either* the control plane or the CPN go down, the data plane will go with it as well.

Orion will use a *hybrid* approach for it's CPN, which will be discussed later.

### Software Design Considerations

The main concern here, is to enable the controller to support concurrent development of features by many small teams of engineers, working separately. This requires high debuggability and modular design. To this end, the following have been imposed over the controller.

- **Microservice Architecture:** The controller is made up of a large number of independent bundles, together creating features that the controller provides to the control plane (this, at least in terms of wording, is quite similar to ODL, which uses OSGi framework for the same purpose, I can only hope it's less painful though)
- **The NIB:** Which is publications/subscription database of messages. The NIB itself is centralized, whereas the rest of the controller is distributed. The main reason for keeping the NIB as a single entity is that it makes things much easier down the road, and this being a single entity, necessarily establishes a natural ordering between events in the network, making the debugging of the control plane much easier and more convinient.


## Implementation Architecture

An abstract view of the controller is provided below:

![[Pasted image 20230110155429.png]]

The data plane consists of multiple switches that run [[OpenFlow]] agents within themselves, allowing for the control plane to program flows, gather statistics and subscribe to notifications from each switch. It bares no mention that the controller uses OpenFlow as it's southbound API.

The controller itself is made up of many different internal applications, including but not limited to the `FlowManager`, `TopologyManager` and `ConfigManager`. The NIB consolidates all these internal applications and exposes a uniform northbound API to all Orion applications (these are the things that engineers may develop separately).

This is very similar to the MD-SAL in ODL, however the key difference is that the API is exposed equally to all external applications, it does not require internal YANG models to be configured for the API to work with the new application (i.e., a new feature need not be installed within the controller, as long as the current northbound API suffices).

### The NIB

The NIB is a database that maintains the current state of intent and networking. NIB entities consist of the following:

- **Configured Network Topology Entity:** The configurational state of the topology graph and it's internal entities. These include the usual suspects (i.e. `Node`, `Port`, `Link`, `Interface`, etc.)
- **Network Runtime State Entity:** The operational state of the topology. Including many things such as forwarding state, protocol state and statistics.
- **Orion App Configuration:** The configuration of Orion applications (this is maintained separately from the network topology).

