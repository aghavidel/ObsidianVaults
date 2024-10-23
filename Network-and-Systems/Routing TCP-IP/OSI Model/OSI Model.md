# Intro.

The Open System Interconnection Model, or simply the OSI model, is a *conceptual model* created to allow system engineers and designers to break every problem associated with networking to individual computers across a possibly unreliable communication medium.

The model was developed to break down this problem into *almost independent* **layers**, and then divide these layers into subproblems that can be addressed by engineers that gained expertise in problems about those individual layers.

It is a clear, concise embodiment of how engineers divide this fundamental problem into several layers of abstraction, and thus proved useful to further the cause that would finally culminate in the design of [[ARPANET]] and the internet.

The model, although a bit too complicated and with arguably a bit bloated, was and still is the main frame of reference when we speak, teach or discuss internet protocols without delving too deep into the nitty-gritties of each of their implementations (which of course, vary considerably between designers, applications and design goals).

While OSI was championed by the International Organization of Standards (ISO) and had a healthy body of well implemented protocols for each of its layers of abstraction, it proved too rigid and was cast aside in favor of the simpler Internet Protocol Suite of networking protocols, designed by the Internet Engineering Task Force (IETF).

While conceptually, it is still as widely used as it was in the first years of its creation, the implementations under the OSI model are for the most part considered a relic of the Protocol Wars over what to use as the foundation of the internet (with the glaring exception of [[IS-IS]], which is still used and isn't going anywhere anytime soon).

That said, no book, lecture or engineer in the right mind would abandon it as a frame of reference for discussion about the internet, and so, at least a bit of knowledge about it is absolutely necessary.


## The Layers

Each layer deserves some note of its own, but in brief:


| Layer Name       | Operation                                                                             | Example Protocols                 |
| ---------------- | ------------------------------------------------------------------------------------- | --------------------------------- |
| [[Physical]]     | Delivery of raw bits over connection media                                            | PCM Specifications / G.703        |
| [[Data Link]]    | Transmission of *frames* of bits between two *nodes*                                  | PPP / LLC / [[IS-IS]]             |
| [[Network]]      | Management of multiple nodes in a network. Functions like routing, addressing.         | [[OSPF]] / [[RIP]]                |
| [[Transport]]    | Enable aggregation of packets into segments. Allow reliable connection.               | [[TCP]], [[UDP]]                  |
| [[Session]]      | Creation/Setup/Teardown of connections. Management of multiple end-to-end connections | [[Sockets]]                       |
| [[Presentation]] | Data formatting/serializing/simple encryption and decryption                          | Some parts of [[TLS]] and [[SSL]] |
| [[Application]]  | Marries networking functions with the OS                                                                                      | [[HTTP]]                                   |


## Services

Each layer must implement a functionality that is described as a *service*. There should be at the bare minimum, two services per layer (with the exception of the top and bottom layers which can get away with just one).

For clarity, layers are numbered from 1 to 7, starting from the physical layer and ending with the application layer just like the table in [[#The Layers]] section.

- The layer number $n$ must provide a service to layer number $n-1$ by exchanging data in appropriate format with that layer.
- The layer number $n$ must provide a service to layer number $n+1$ by allowing for data exchange between the two. The lower layer may assume that the higher layer can format the data by itself or may also implement appropriate formatting by itself.

Aside from these, each layer can have internal services that act as a subroutine to the services above, allowing engineers to breakdown these services into smaller micro-services that can be chained together in interesting ways.

At each layer, the data format(s) that the layer can process and comprehend are called a *Protocol Data Unit* or a PDU. Each layer is able to pass a PDU one layer lower by appending to the PDU a scarp of data that is called a *Service Data Unit* or an SDU.

![[Pasted image 20220913192042.png]]

These SDUs are more akin to packet headers, as they essentially provide some instructions and specifications to the lower layer that allows that layer to process the PDU according to the needs of the protocol and the service.

The process continues until the fully padded and serviced PDU reaches the lower physical layer and is then transmitted across the internet. So essentially, each node that implements the OSI model guarantees that the appropriate implementation of each of the 7 layers has been done and thus communication may ensue.


>[!IMPORTANT]
>One main principle of the OSI model, is that all of these layers must be designed as independent as possible.
>
>To express this better, the design of each of these layers must be in a way that allows *any* layer to be changed in terms of its implementation, while preserving its functionality.
>
>That functionality ends up being whatever the layer refers to as a *protocol*. So the main constraint of the OSI model is that:
>> As long as the protocol remains unchanged, the implementation does not matter.



### Cross-Layer Services

There are services that by nature, do not really fit into the layered design of each service in the OSI model. The most notable example is security services.

Each security service must design itself around the principals of confidentiality, integrity and availability which in practice, does not lend itself to the layered design of the OSI model and thus ends up being a service that is implemented across several layers.

This adds considerable amount of complexity to these services as a result. 

These include things like:
- Transport Layer Security ([[TLS]]) that is implemented from the session layer all the way up to the application layer.
- Pretty much every wireless communication specification must consider cross layer design between its physical layer and its data-link (or MAC) layer. This is the result of the everchanging state of the communication medium in a wireless network, which wired networks don't really face that much.
- [[MPLS]] is also a cross-layer protocol, it essentially operates between layer 2 and 3 at the same time, in return it is able to indeed, perform over multiple protocols at the same time.


## Legacy

While the OSI model has the benefit of being precise and accurate in its description of networking methodologies, it was also complicated, inefficient and according to some, unimplementable.

The OSI evolved with TCP/IP and so they had to compete with each other to find widespread use in the industry. The problem was that:

- Those who had already adopted TCP/IP were resisting the change from their current model to OSI, simply because they didn't consider OSI flexible enough for their networking needs.
- Those who wanted to implement OSI, were unsure what kind of *implementation* they should use. Bear in mind that the OSI model only specifies *methodologies*, not *implementations*, combined with the complexity of most OSI specifications, different implementations were rarely compatible.

Another big factor in TCP/IP winning against OSI was that it was TCP/IP that ended up being implemented in UNIX, the operating system that for the most part, governed the main body of [[ARPANET]]. So these engineers had much less trouble building networks over TCP/IP compared to OSI, simply because most of the work had already been done for them.


## Comparison With TCP/IP

It is accustom to create a toy mapping between TCP/IP and OSI to avoid confusion when talking about TCP/IP protocols within a context that uses the language specified in OSI.

A widely adopted mapping is the following:

![[Pasted image 20220913203424.png | 500]]

Some may also say that the application layer of TCP/IP encompasses all of the 3 upper layers of the OSI model. 

In practice, most engineers and computer scientists really do not make concise distinctions between these 3 layers, probably because the fifth and sixth layers in the OSI are very "thin" compared to their other siblings, and contain much less of an actual specification.

For the most part, anything that isn't specifically in the transport layer, but uses some operating system services, is considered an application layer creature.

