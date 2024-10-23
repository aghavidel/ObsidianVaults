**Paper: [Running BGP in Data Centers At Scale](https://www.usenix.org/system/files/nsdi21-abhashkumar.pdf)**

We now consider *how* one would actually route the Data Center itself. We have considered the techniques (e.g. ECMP and TE and whatnot) but the exact mechanism that enables it has not yet been discussed here.

There are 2 broad categories of routing schemes, both in the Internet and the DC:
- Decentralized, with the prime example being with BGP as Facebook did (this paper)
- Centralized, with SDN (as Google did with Orion, the next paper)

# Facebook BGP

Facebook, aware of the two choices, adopted NOT to use SDN. The reasons being:
- SDNs were already hard to scale, since it was noted that a controller, handling thousands of switches, won't be able to react to failures or topology changes quickly. Google also had this problem, but they solved it by throwing money at it (we'll see it later)
- Second, Facebook was quite pragmatic here. BGP, despite all of its problems, **scales very VERY well**, whereas no SDN controller exists that can heavily scale yet (even Google's Orion has its limitations). Facebook decided not to go that route, since it would necessarily entail that they would have to create the SDN stack themselves, and that would be a headache and slow them down.
- BGP has been standardized over the years, meaning that even if they built their own in-house switching appliances, the routing stack wouldn't change, which means that the DC can be upgraded by just throwing better hardware and money at it. This would be useful, since Facebook did eventually make their own virtual switching software, [[FBOSS]] (we'll come to this later!)

It is also notable that Facebook used a **Pure BGP (eBGP that is)** system, no IGPs or anything like that was needed (in contrast to Google's Jupiter that did use OSPF at least). Facebook had doubts about the scalability of IGPs (and those doubts are reasonable, IGPs are very chatty and don't scale anywhere near as well as BGP).

## Basic Facebook DC

Here is a view of it:

![[Pasted image 20240214115446.png|600]]

Facebook also used the classic DC hierarchy, that being Spine, Fabric and Rack. Like Google, Facebook also uses at least a 2 stage fabric layer, called a **Pod**.

Facebook seems to have employed many pods, each running FBOSS and each pod serves 48 servers. Racks of course have their own ToR switch and they have as many fabric switches in a pod as there are spines.

Facebook also adopted the 4-color design that we saw with Google for failure resilience.

- Up to 16 fabric switches (FSWs) may be in a single pod
- Up to 48 rack switches (RSWs) may be in a single pod
- RSWs and FSWs form a complete bipartite graph
- Thus at most 16 spine switches might have to be deployed as well

## Routing Design

As we mentioned, Facebook used no IGPs and even iBGP. Each switch is its own Autonomous System (AS) and peers with its neighbor with eBGP. Of course, this means that the routing table starts to get very large as you get more and more switches, so the first challenge is to actually scale down that table!

>[!REMINDER]- Reminder on BGP
>We'll just go over what we need.
>
>In BGP, every node is assigned an AS Number (ASN). We won't discuss iBGP and focus only on eBGP, so each pair of nodes will have different ASNs. 
>Each node, contains a list of IP prefixes (i.e. an aggregate of IPv4/IPv6 addresses, like `20.10.12.0/24` for example), and BGP will distribute the *reachability information* of each prefix, which is a tuple in the form of `(next hop, destination prefix)`.
>
>BGP also supports a very wide range of policies, which means that nodes can:
>- Reject certain advertisements
>- Manipulate how an advertisement goes forward into the network
>- Deprioritize an advertisement (e.g. for backup routes)
>
>Each prefix announcement also updates a local data base in each node called the Routing Information Base (RIB). This database maintains not just the next hop, but also the *AS Path* attribute, which is the list of ASNs that need to be traversed to reach the prefix.
>One function of this is to prevent loops. A RIB will reject an advertisement if it sees its own ASN in the AS Path.
>
>![[Pasted image 20240214120956.png|200]]
>
>BGP scales very well, but when changes occur in the network, BGP can take a significantly long time to converge. BGP's Delayed Convergence has been extensively studied.
>
>Finally, BGP offers this feature called **Confederations**. A confederation is a set of *Confederation Members*, each with distinct ASNs, but the confederation as a whole, appears as a single AS with its own ASN. 
>
>![[Pasted image 20240214122843.png|600]]

The main design principals of Facebook for its routing infrastructure is:
- **Uniformity of Configurations:** Try to use the same configuration as much as possible. This makes it much easier to cope with network changes and keep backups.
- **Simplicity:** Minimize BGP features that are needed. Ideally just stick to eBGP.

Facebook defines a notion of **Peer Groups**, where a group switches that share the same role and pod are grouped into a peer group. So:
- All FSWs in the same pod are a peer group
- All RSWs in the same pod are a peer group

One invariant of Facebooks design is that **Anything in the same Peer Group SHOULD have the same configuration parameters. They may only be differentiated at most in an IP address to differentiate neighbors.**

Facebook adopts the following ASN schemes:
- All Spine Switches (SSWs) in the same Spine plane have the same ASN as a BGP confederation
- All FSWs and RSWs have different ASNs, but those within the same pod would be managed under the same confederation (so to the outside world, the whole pod looks like one AS)
- Every pod is a confederation 
- All ASNs can be reused in all other pods in all other DCs

![[Pasted image 20240214122228.png|600]]


To make use of the path diversity in their topology, Facebook modified their BGP implementation to always use ECMP when forwarding to the next hop. The clever ASN assignment scheme helps immensely here.

Now, off to the IP assignment ...