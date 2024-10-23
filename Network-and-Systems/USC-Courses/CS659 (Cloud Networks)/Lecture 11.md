
**Paper:** [B4: Google's Software-Defined WAN](https://dl.acm.org/doi/10.1145/3230543.3230545)

# B4

We have discussed DCNs at length, now, we turn our attention to a much more geographically wide network, aptly referred to as a **Wide Area Network (WAN)**.

![[Pasted image 20240214171411.png|500]]

Each site above refers to a group of buildings around the same geographical location. These sites can be very far away. Google for example has sites in Oregon, LA, DC and many more.
The (private) network that connects these sites, funded directly by Google itself, is a network run directly by Google without any ISP involvement. 

So first question, *why even do this?* Why not use a commercial ISP network?
- On an ISP network, you MUST keep it under-utilized (as low as 30 and 40 percent) to make headroom for accommodating failures.
- When a failure happens, all traffic is treated the same.

Problem is that these WAN links that go long distances are extremely expensive. Having low utilization for them would be very VERY wasteful. Thus, it makes sense that Google would build its own network.

B4, when the paper was written at least, looked like this:

![[Pasted image 20240324214024.png]]

Google has two WANs, B2 and B4:
- **B4**: Serves only Google and routes inter-DC traffic
- **B2:** For user requests and responses

>[!FAQ]- There Was Also A B2?
>Google has another WAN, B2, that faces the public via other ISPs. B2 would have much less utilization as a result, and as such, Google put most of its own traffic to go through B4, so that B2 becomes smaller and much more cost-effective.

The traffic that B4 handles is spread across only first-party applications of 3 kind:
- Replicated user data across campuses (Large traffic, not too critical)
- Applications accessing data in other campuses (Not very big, but pretty important)
- State synchronization between distributed applications (Small, but very high priority)

It is important to note that in this network, the largest flows are *elastic*, which means it tolerates delays and thus if a failure lowers capacity, we can delay them.

>[!FAQ] How Does Google Prioritize Traffic
>Google relies on the IPv4/v6 Differentiated Service fields (`DiffServ`). So they tag very high priority traffic directly in the IP header.
>It is safe to do this in a WAN here, since Google owns the whole network and no one can cheat by setting the header bits.

So how does B4 make use of all of these properties?
B4 exploits the following facts:
1. Elasticity of high volume flows (and again, they can only do that because they own the whole network and its applications)
2. Small number of nodes
3. Full control over the applications and the traffic they generate

The main elements of this design is that:
- It uses hierarchical SDN 
- WAN routers are made out of Merchant Silicon
- It supports centralized TE via the SDN

For the router, they use a Clos topology to make large switches:

![[Pasted image 20240214174653.png]]

Each of these routers, run an OpenFlow agent and is made out of the same Merchant Silicon chip that made up Saturn and Watchtower. Things are probably changed now, since the B4 paper pre-dates the introduction of OCS into Jupiter.