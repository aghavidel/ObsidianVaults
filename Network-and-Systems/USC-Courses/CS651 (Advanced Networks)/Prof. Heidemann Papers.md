This document is a summary of papers for the CSCI 651 course taught by Prof. John Heidemann, which has a different syllabus compared to the one taught by professor Govindan.

# Design Principles

These papers cover the main design principles of the internet during the past 4 decades.

## DARPA Internet Protocols

The DARPA project was an attempt to connect the ARPA Wide Area Network (aka the ARPANET) and the ARPA Packet Radio Network (yes, they had packet radio networks even then!), and while doing this, certain design principles started to take shape, these being:

- **First Level Goals:**
	- Backwards compatibility and incremental deployment, since the network was too large to fundamentally change
	- Packet Switch instead of Circuit Switch, since the majority of the applications being used gelled well with that mode.
	- Store-and-Forward packet processing, which meant that header data and TLV processing were needed for defining protocols.
	- **Survivability**, the most important goal
- **Second Level Goals:**
	- Distributed management 
	- Support different protocols, don't ask too much from the clients
	- Be cost effective (a goal that was not met)
	- Make it simple for clients to attach to the network with minimum devices (also not met)
	- Resource accountability (you need this for billing for example)

### The price of survivability

To be robust against failures, the End-to-End design is used. The reason is this:
- Assume two endpoints are communicating using a network. The ongoing communication would require storing some state *somewhere*, that state can be the state of the transport (number of ACKed packets, SEQ number, and others).
  All of this data needs to survive failures for the service to continue. We want to make sure that only in the case of a complete partition between the hosts (i.e. no path existing between them) will the service break down.
  However, since the nodes on the path can breakdown arbitrarily, we must then *replicate* the node states in the networks.
	  - Replication makes sure that some number of failures can be tolerated, however, what to do if even that number is exceeded?
	  - Replication requires synchronziation among nodes, and that is very difficult
- For these reasons, DARPA did not use this approach, instead all communication state was moved to the hosts, and this created the concept of **Fate Sharing**, which means:
  
>[!IMPORTANT] Fate-Sharing
>Two nodes, or services, or entities, are said to "Share Fate", if any failure in one will cause an immediate or otherwise deterministically correlated failure in the other entity as well.

Fate-Sharing is a good thing with end-to-end designs, since it guarantees correctness as long as end-host algorithms are correct (this is why if TCP performs correctly, we can infer that it performs correctly in any network). However, it does NOT follow that it is necessarily performant (more on that later).
The main prize of Fate-Sharing is:
- Easier to implement
- Correct, until full partition

However, there are also problems:
- Nodes between hosts must be stateless and thus oblivious to certain things
- End-host algorithms must be correct an thus some notion of trust between hosts are needed

### Types Of Service

Internet must support different types of service and thus should not ask too much from clients for basic operation.
This was seen with the TCP/IP stack. Originally, it was not even called TCP/IP, it was just **IP**, and TCP was a mode of reliable transportation over the unreliable "Datagram" mode that IP provied by default.

However, TCP was soon seen to be not effective for everything, examples being:
- Realtime digitized speech signals, where some loss or corrupted packets are tolerated, but delays (which TCP can have) are not
- Distributed debuggers, where it is most important to connect to nodes easily without a handshake being needed

It was for this reason that the IP and the TCP stack broke apart, and the IP layer was to only provide a *Best Effort* service, and if reliability or other things were needed, the application layer must make up for that using selective ACKs. 
To provide a basic application interface, a Datagram protocol was designed for users, aptly named the User Datagram Protocol, or UDP!

# End-to-End Design Principle

We have seen this principal above, but stated concretely, it means the following:

>[!IMPORTANT] End-to-End Principle
>All network functions can be correctly and completely implemented only with the help and knowledge of the endpoints of a communication. As such, such service cannot be provided as an in-network feature.
>However, an incomplete version of the service can be implemented in-network as a performance enhancement.

The main reason for this argument was seen above. If in-network state is used, then replication is necessary, and replication only lives up to a point and can break down far before full partition happens.

# Routing

Here, we discuss scalable routing in the internet, which really is just BGP.

## BGP Route Policies

We have talked about [[BGP]] at length, and how it evolved from the original GGP, to EGP, then BGP, and now we have BGP version 4, which is the only BGP version we really care about, and the main reason for that is that this is the protocol that implements Routing Policies.

Implementing what ISPs want requires flexibility, not just being a correct path-vector protocol like what the original BGP was.
To this end, ISPs changed the original BGP to suit their need, however that also meant that ISPs would have a very hard time imposing and regulating their communication with each other, when they were literarily speaking with different protocols.

To this end, BGP has evolved from its original design, into a beast that gives many options for changing how routes are advertised, not how they are calculated (we are way out of the realm of shortest path routing now, we are not trying to connect more points on a graph, in fact, we are trying to **limit** it, and make sure things happen as we want them).

BGP route policies are separated into 4 categories:
- Business relationship policies, which describe how neighbors are defined and how they are treated
- Traffic engineering policies, preventing congestion and controlling how traffic flows over peering points
- Scalability policies
- Security policies

![[Pasted image 20240409021455.png|500]]

### BGP Decisions Process

The main thing in the BGP protocol that can be tuned, is the decision process. The process where we receive paths going to a prefix (i.e. an address and a mask, say a path to `68.181.218.0/24`) from multiple sources and most decide which one to use.
To this end, the decision process of BGP uses multiple route attributes, which can be set by the users if needed.

By default, BGP tries to pick a route with the least number of systems to traverse (so a minimum hop path where each hop is an AS), and somehow break the tie if multiple routes survive that process.

These requirements are:

![[Pasted image 20240409021846.png|500]]

Here
- `LocalPref` is just a number that can be set by the user. It makes sure that the user will have the final say in which local routes are to be accepted. For example, static routes would have the highest local preference.
- The `MED` is the Multi-Exit Discriminator, where two peers that have multiple links between them, can use it to dictate which link must be used (this can implement hot/cold potato routing)

Beyond this, the set of routes that get into the decision process in the first place can be filtered using **Import Policies**, which can describe filters on routes (so for example, don't route prefixes in the range `63.8.0.0/16`), and also the output of the decision process that needs to be advertised can also be modified through **Export Policies** (so for example, do not pass routes with more than 5 hops to a neighbor).

Operators may also use **tagging**, adding essentially variable length strings to routes that can have meaning only among certain ASes and are ignored by others. The prime example of this is the BGP **community** tag that essentially when detected on a route, invokes a local function in the AS that can affect the decision process for that route.
For example, the function can affect how the router sets the `LocalPref` value for that route, or prevent it from being accepted in certain stages of the decision process.
It is highly expressive, but non-standardized and as such can be prone to misconfiguration. 

Most of these policies are implemented with **route-maps**, which is just a match-action mapping from BGP attributes to an arbitrary function that receives a route and spits out a changed route.
This can be used to implement both inbound and outbound traffic engineering (although it is difficult and a bit unpredictable).

