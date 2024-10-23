# PCEP Primer  
  
The Path Computation Element communication Protocol (PCEP) is a protocol that is used to establish synchronized connections between a PCE and a PCC. This note describes the operations of PCEP as well as the PCE itself. 
The note assumes basic understanding of traffic engineering and knowledge of IGPs.  
  
## The PCE  
  
As defined in [RFC 4655](https://www.rfc-editor.org/rfc/rfc4655.html), the PCE is "an entity that is capable of computing a network path or route based on a network graph, and of applying computational constraints during the computation".  
  
So simply, PCE is an application that consumes topology data (usually in the form of TE databases) with a set of constrains advertised by the client, and computes the best path over the topology that satisfies the set of constrains or informs the client that such a path does not exist.  
  
There is a wide range of constrains that PCE can consume, most of which are provided by RSVP (so bandwidth and capacity constrains at the very least). So for example, the PCE can create TE based LSPs that can be pushed into the network, and guarantee that bandwidth is not totally consumed along the path given the current state of the TE database.  
  
### Preface: What PCE Actually Is and Isn't!  
  
PCE is a VERY powerful tool, yet care must be taken when using it. It is important for an engineer to know what exactly this PCE can do or shouldn't do. We'll go over this point to point.  
  
1. PCE is NOT "an all-seeing oracle in the sky", in this sense, it is by no means supposed to completely tear out any amount of intelligence from routers that use fundamentally distributed protocols for path computation. It also should NOT be thought of as some sort of SDN based solution, as it does not intend to decouple the control plane from the data plane by any means. More on that later.  
2. PCE is an extension of the same technique used within traffic engineering, but in a centralized manner, giving more control to the network engineer over what configurations should be pushed into the network (it also allows it to be done much easier).

So in brief, it is merely a tool, it is not an enhancement of the already present network protocol stack by any means, and it does not really add anything to the already available networking intelligence distributed into the network.

#### Where PCE Helps

There are many situations where PCE is quite a reasonable solution though:

- **CPU Intensive Path Computations:** Routers simply do not have the resource needed to do most path computations most of the time, these computations are especially demanding in the case of things like objective minimizing LSPs or computing [[Steiner Trees]]. A PCE however is able to support dedicated hardware for such purposes and be much more efficient for such requirements.
- **Multi-Domain Visibility:** In the current day and age, the internet consists of many individual, independent blocks of networks, which do not share traffic engineering information with each other, thus a single router will be clueless about what is happening in other domains, unless TE data is advertised outside of the destination domain, which gives rise to many concerns about confidentiality and security of such information, since these domains must remain mutually suspicious of each other.
  A distributed network of PCEs can solve this problem (though there are nuances about *what* they can share with each other that preserves security), or if security is not a concern (somehow!), the information can be aggregated into a single, all-encompassing PCE server that does the job for all domains.
- **Maintaining TE databases is expensive, or not possible:** Maintaining dynamic, large databases like TEDs is a resource heavy task, not necessarily feasible for a router, similarly, not all routers may have appropriate [[IGP]] extensions for TE. In such cases, the PCE can be used to build a TED and maintain it without putting too much strain on the data plane.

#### Where PCE SHOULDN'T Be Used

Broadly speaking:

- PCE is not applicable to the internet as a whole despite being a very powerful tool. PCE itself is merely applicable to set of domains with *known* relationships, like how ISPs connect with each other.
- TEDs do not necessarily reflect the true state of a network, as they do not take into account certain resources that are necessary for successful LSP establishment. Hence, in use cases where successful LSP establishment requires hard guarantees, PCEs are not the solution.


### PCE Architecture

PCEs can be utilized in different architectures in relation to how they communicate with networking appliances or how they communicate with each other (if there actually is a need for that at all). We won't go over all of them, just the important (and prominent!) ones that have been somewhat standardized.

- **Composite PCE:** PCE is built *into the router* , meaning that the PCE has direct access to the data plane like the following:
- **External PCE:** 



