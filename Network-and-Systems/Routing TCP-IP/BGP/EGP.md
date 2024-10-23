# Intro

The *Exterior Gateway Protocol* was a *Neighbor Reachability* protocol that allowed routers to keep information about their neighbors, and with that information, they allowed network operators to exchange reachability information between major points of presence.

It was designed to overcome the major scalability problems of its ancestor, the [[GGP]].

It was designed by Eric Rosen under [RFC 827](https://www.rfc-editor.org/rfc/rfc827): and paved the way to the creation of what would ultimately become [[BGP]].


## The Main Design Constraints of EGP
Eric Rosen detailed the major design constraints of his protocol over GGP in the RFC document:

- The network depends on too much trust between gateways. A malicious participant in the network can easily wreak havoc on the current network configuration by just shouting nonsense into the network. 

  >The network must be designed in a way that asserts that these gateways remain "mutually suspicious".  

- As the network grows, the probability of topology change at any moment increases. 
 
  With GGPs current implementation, a network change at the right place and time can flood the network with excessive update messages and disrupt the steady  flow in the network. 
  
  >The overhead of such a greedy routing protocol is simply too high.  

- The number of administrators operating in the network increases with more gateways added into the network. 
  
  People can be difficult to work with, more specifically, people might be unable or flat out refuse to update their software. 
  
  >The implementation of routing protocols becomes more and more inflexible, since any change to the routing protocol must  either leave a "back door" for older protocols to keep doing their job, or force itself on every single gateway participating in the network.  

- People implement the same thing differently, and as mentioned before, resist changing things that are already working. 
  With more and more implementations of GGP flooding the current network stack, as Rosen puts it: 
  
  >It becomes impossible to regard the internet as an integrated communications system.  
  
Rosen proposed that [[ARPANET]] moves from a single entity, into a network of self-governing domains, he actually coined the term "Autonomous System" and its 16 bit identifiers at the time. 

It was suggested that ASNs should be assigned by IANA (the same entity that assigns IP address blocks) and some of them be reserved for private use (once again, like IP addresses). 

Note that these days, we have graduated from 16 bit ASNs to 32 bit ASNs for the same reason that we created IPv6 after IPv4.


EGP by design, was never *actually* a routing protocol. It was merely a mutual language that gateways could use to speak with each other. While it may look like that EGP would be useless without this, bare in mind that it was just a temporary solution to the scalability problem of ARPANET, and yet it still shares much with BGP.

Another important note, is that since the internet was not particularly large in those days, the path calculation and routing could be delegated to whatever was used as an [[IGP]] (either IS-IS or OSPF), these days however, distributing exterior paths into the interior routing process, almost always kills that process, since the number of routes is just ridiculous.


One important note about EGP is that it was designed to be *point-to-point*, so even neighbors need not be directly connected.

For example, here you can see an example:

![[Pasted image 20220910180518.png|500x00]]

In the topology above, the two routers using EGP are not directly connected, instead there is 2 hops of [[RIP]] between them.