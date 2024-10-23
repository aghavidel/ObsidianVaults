# Intro.

MPLS or *Multi-Protocol Label Switching* is a forwarding practice used over IP networks, with the main motivation being to make the data plane faster and more programmable. It is for this reason, a kind of cross-layer technology between layer 2 and 3 (Network and Data Link).

MPLS has found use outside of just forwarding, and of course is the centerpiece of [[MPLS-TE]], which builds on top of it. It also plays significant roles in more modern routing practices like [[Segment Routing]] which also uses similar concepts.

## Labels

The main unit of data for MPLS forwarding is *labels*. Labels are often 20-bit integers assigned as packet headers that define the path that the packet must traverse once received by a router operating on that path (or often, the phrase *tunnel* is used for these paths, as we are still technically using IP paths, but we are "tunneling" through multiple of them to make more complex paths).

How these labels are utilized, depends on what mode MPLS is being configured, there are two modes, there is Frame Mode and Cell Model.

Cell Mode almost always concerns using ATM, so we'll skip that for now, just know that it exists. Let's discuss the MPLS Frame Mode, which is the main thing that you will see used.

### MPLS Frame Mode

In frame mode, labels are prepended to the packet, in front of the layer 3 header. Labels are usually  20 bit values, so we can have just over one million labels active in each independent domain. Labels are directly added to frames, meaning that special hardware will need to be present to allow for MPLS fabric to be used (these are called Label Switched Routers or LSRs).

Frames may contain multiple headers in a label stack, at each hop, the top of the stack only will be under consideration by the routers. 

>[!NOTE] Label value vs Label Stack Entry
>Label is a somewhat deceptive term here. While in conversation, label usually only refers to the 20 bit value associated with it, in practice, it is accompanied with 12 other bits, making it a 32 bit *label stack entry*, the accompanying bits are:
>- **EXP:** Three bits reserved for experimental use, usually reserved for QoS identifiers.
>- **S:** A single bit signifying the bottom of the stack.
>- **TTL:** The usual TTL value, most of the time (definitely not always though) a direct copy of the TTL value in the IP header.
>
>It is this 32 bit string that is prepended to the frame, not the label value by itself.


### Forwarding And FEC

When using simple IP networking:

- Once a packet enters a router, the destination of that packet that is encoded in the header is inspected.
- The value of the destination is looked up in the local FIB of the router, mostly using a longest prefix match.
- If a hit happens, the router switches the destination to the next hop, sending the packet out, if no hit happens and no default route is set, the packet is dropped.

With that said, destination usually is NOT the only thing a router can consider for routing a packet. In many simple networks, routing can be done by matching on the *interface* that the packet has arrived on, or in our case, based on the MPLS label value.

Depending on how a router is configured, two packets can be routed in the same manner, thus it is useful to classify packets into larger groups (compared to the large amount of destination addresses possible) that instantly recognize for us, how those packets should be routed.

This class, is called a *Forwarding Equivalence Class* or FEC, and can be expressed in many different ways, including MPLS. In MPLS however, the FEC itself is mapped into a label value, meaning that after the FEC is determined and the label is assigned (given of course that the labels are configured consistently throughout the network), the string of labels that the packet must traverse are immediately determined, and thus routing is done much more easily.

The main benefit of MPLS is that it greatly reduces this classification overhead by essentially precomputing the bulk of the work. In MPLS, an egress router (usually called a LER), receives an IP packet, it then:

- Determines the FEC of this packet using available data (mostly only destination IP, but it can involve port numbers, interface names, etc.).
- Once the FEC is determined, the router checks whether or not that FEC has an associated label, if it does, the label is **pushed** on the packet and it is sent on the appropriate interface (so the FIB entries of MPLS map into pairs of `(interface, label_value)`).
- After this, all intermediate routers simply **swap** the label, which means that they inspect the label value and correspond it to the value in their own FIB, then the top of the stack is removed and the associated label is added.
- At the egress, the routers simply **pop** the final label and send the packet to the receiver in that network like always.

This shows the 3 basic operations of MPLS; *pop*, *push* and *swap*. Given these, MPLS need only rely on the control plane to distribute information about what FEC maps into which label. These protocols include many well know protocols like [[BGP]] and [[RSVP]], but they also include a Label Distribution Protocol or LDP which we'll discuss later.

