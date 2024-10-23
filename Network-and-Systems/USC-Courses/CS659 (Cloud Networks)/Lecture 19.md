# Network Virtualization in DCNs (cont.)

Continuing from where we left off. At the time where the VMWare paper was written, most cloud providers did not have any support for network virtualization. The problem in this frame of thought, becomes "***How to take a private network configuration and port it into a public cloud?***".

This port should:
- Preserve IP addresses
- Preserve network functions (e.g. firewalls)

The virtualized network should *look* exactly like the original enterprise network that it was based on. Getting inspiration from how VMs are designed, we can propose to create a **Network Hypervisor**, that looks at the original physical network specification and makes a virtualized version of it on a physical infrastructure.

![[Pasted image 20240327162120.png|300]]

We want this Network Hypervisor to support:
- Different topologies (the original network could be a huge L2 domain or normal L3 network)
- Should support on-path processing (Access control or NATs)

## Network Hypervisors

![[Pasted image 20240327163310.png]]

Network Hypervisors provide a packet abstraction. They run a piece of software on large and capable Linux boxes that mimic switches. The most important thing that they should provide is consistently implementing the ***logical Datapath***.

>[!DEFINITION]
>The sequence of actions that a packet is subjected to upon ingress to a switch is called a Logical Datapath. 
>For example, in a layer 3 network we could have a logical Datapath consisting of a pipeline of `ACL -> L3 -> NAT`, where we first check the IP source, then we lookup the next hop and then do NAT.

As for how such a software really works:

![[Pasted image 20240327163844.png|500]]

There is a lot to unpack here:
- The source VM sends a packet out of its virtual NIC, destined to the other VMs IP address.
- The hypervisor, **which is running in the source box**, receives the packet on physical Datapath (e.g. a Linux box running OVS).
- The hypervisor has programmed all the tables on the OVS instance to mimic the Logical Datapaths that were inferred from the network configuration (again all of this is done in the source!)
- The OVS instance subjects the packet to its table and does exactly the same thing as the physical network would. It then emits the packet towards the destination virtual NIC that goes to the other VM.
  On the last step, the packet is encapsulated with the IP address of the destination physical IP address.
- In the last step, the packet header is opened and passed to the virtual NIC. The original packet listed the IP of this virtual NIC as its destination, thus the VM will accept it and the process is complete.

As for what actually prepares all of this, the application that VMWare designed is the **Network Virtualization Platform (NVP)**.

NVP will receive the virtualized network configuration (which we won't discuss its format or how it's made).
- NVP will parse the configuration and create tables for each logical Datapath
- The cluster will spawn VMs for each compute node and assign them to hosts in the cluster. The cluster internal networking should make sure that all hosts can reach each other.
- Virtual NICs of the VMs are assigned the appropriate IP addresses and then connected to the OVS instances within each host that does the table processing
- The tables are inserted into the OVS instances using a SDN controller

>[!FAQ] Broadcasts and Multicasts
>The physical network that we are virtualizing might attempt to do multicasts or broadcasts to other nodes on that network. How would we virtualize these?
>In the IPv4 definition of multicasts, IPs that follow after the `255.0.0.0` range are multicast groups. The physical network will use those and so NVP will know what are multicasts (similarly broadcasts).
>
>![[Pasted image 20240327171037.png|500]]
>
>To do broadcasts/multicasts, we use **service nodes**, pieces of software in their own cluster (we need this to make it scalable and to load-balance) that receive a packet to some destination group and then replicate it and send it to VMs. This would also have to be encoded into the OVS tables.
>Additionally, packets that are destined for outside of the network are passed to the cluster *gateway*. The source of the packet that goes to the outside world will be the **external IP** associated with VM(s), and the cluster gateway will encapsulate it upon return and pass it to the destination VM.

>[!FAQ] How To Lookup The Next Table?
>OpenFlow implements a `metadata` field for flow matches that can contain the next table you will go on the processing pipeline. NVP will use it to specify which table should be used next.

>[!EXAMPLE] Implementing An L2 Switch
>A L2 switch can have the logical Datapath like the following:
>
>![[Pasted image 20240327172728.png|400]]
>
>The trick here is that we need another table that forces packets to go through the first ACL. We call this the `map` table that adds the table ID of the first ACL table as a metadata to the packets. This can be needed for other packets as well. Same thing might be used for subsequent tables as well.
>
>![[Pasted image 20240327172927.png|400]]
>
>If source and destination are on the same server, an optimization can be to skip the encapsulation steps entirely.

## Performance Considerations

### Segmentation and Reassembly 

If IP packets are large (larger than MTU) and features like Jumbo Frames are not supported, then kernel will break packets into segments and reassemble them in the end. 
Doing this on the same CPU is very expensive, so many NICs have dedicated ASICs or hardware to do this.

Usually, NICs provide two types of hardware support for this:
- TCP Segmentation Offload (TSO)
- Large Receive Offload (LRO)

For now, forget the second one, focus on the first one.

For TCP, if packets are segmented, header data needs to change (ACK numbers, SEQ numbers or packet lengths). For IP headers, things are easier but modification is still needed.
This is where we have a problem. **This approach cannot work if you have an additional tunnel header in front of the IP header**.

Why?
NICs usually expect TCP headers to follow directly after IP headers. Thus if you try to do reassembly on the NIC, it will get confused for tunnel packets and spit out nonsense. 

This is very annoying, since doing segmentation and reassembly on the CPU is just not possible. Thus, a hack was proposed called **Stateless Transport Tunneling**. We insert a fake header that contains the actual logical destination IP address:

![[Pasted image 20240327174108.png|500]]

### Forwarding State Computation

NVP uses an SDN controller for installing tables on switches. During the early days of SDN, there was a distinction between *reactive* and *proactive* SDN, but now we only have proactive SDN and NVP also just uses that.

![[Pasted image 20240327174451.png|700]]

The controller receives:
- Tenant configuration information
- Location of VMs (i.e. the virtual NICs) and MAC addresses

The problem here is that the output state of the controller is quite large, and the users do wish to change the state of the network often (moving VMs is a good example). Re-computing the state has too much overhead, and thus a better way would be to think of the required change **incrementally**.

NVP creates a domain specific language for this called *Datalog*. We will finish this discussion in the next lecture hopefully.