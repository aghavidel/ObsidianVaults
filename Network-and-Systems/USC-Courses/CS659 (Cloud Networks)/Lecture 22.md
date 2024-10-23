# AccelNet (cont.)

We discussed that processing packets from VMs is slow. 
- The VM needs to transfer a buffer from itself to the Hypervisor
- The Hypervisor needs to transfer the buffer to the kernel
- The kernel needs to coordinate with the NIC to send the buffer

A lot of work, but we did see that SRIOV can skip all of this down to the last step, but we also saw that SRIOV won't work in the cloud.
The final idea that we settled on was to implement all packet processing (i.e. the virtual switch itself), directly in the NIC and then use something like SRIOV to give packets straight to the NIC:

![[Pasted image 20240424123230.png]]


## Background: Virtual Filtering Platform (VFP)

We have seen that VMs have a lot of complicated processing to do with packets. They have NATs, ACLs, export/import policies and all other weird thing that just cannot be implemented easily in hardware. So, all of these are done in software.

VFP has its own [paper](https://www.usenix.org/system/files/conference/nsdi17/nsdi17-firestone.pdf), check it out if you want, but in brief: 

VFP, is Azure's version of such an interface, think of it as a beefier version of OpenFlow and a more scalable implementation than Open vSwitch. It has two main differences compared to what we know these things can do:
- **Explicit support for multiple controllers.** OpenFlow allows multiple controllers for a switch, and Open vSwitch implements some form of support for it. However, there is still significant coordination needed from the control plane itself between the controllers to make it safe. VFP handles a bit more of that, with the goal of minimizing cross-controller dependencies.
- **It supports a stateful Match-Action-Table.** Which means that it can consider the connection state as a whole to make decisions about packets, not just the header of the packet itself.
- **Allows for definition of arbitrary Match-Action pairs.** Similar to P4, but much less generic, there is no need to assume completely arbitrary header format that we should define for the software directly before it can do anything.

Going back to AccelNet, the main Ace of it is that it implements VFP *in hardware*, in particular, it implements VFP in Smart NICs! More specifically:

![[Pasted image 20240424125013.png]]

1. It maintains a software in the kernel stack that is coordinated by a user-space process that actually implements a version of the Match-Action Table that the hardware can understand.
2. When the hardware Match-Action Table misses a packet, it is given to the VFP software in the kernel to decide what the match and action should be.
3. Upon getting the updated version, we come back to the hardware and route each subsequent packet without trapping into the kernel again.

As you can see, it is very similar to how Open vSwitch uses a kernel Datapath and `vswitchd`. This effectively gives it the same performance of SRIOV while being much more flexible. Additionally, it allows the design to:
- Maintain 40 Gbps throughput, and up to 100 Gbps with beefier NICs.
- Allows one to define new SDN actions.
- Allows for **Hitless** upgrade (i.e. you don't have to recompile OVS when you define a new action :/)

The only trouble is the hardware here. It is the same song-and-dance that we had when using P4, we need some hardware that supports Reconfigurable Match Tables (RMT) and has significant memory, since whatever match table we get out of this process can be very large and may not fit in memory.

If you remember, we noted that we could either use FPGAs, ASICs or a CPU. In P4, the choice was made to use ASICs. This means that whatever hardware comes out of this, while it is expensive, will have a long shelf-life since it can handle whatever job we give it efficiently.

However, ASICs are not particularly programmable, and so we need to add CPUs to it to make it programmable, and that also means that the nice and powerful ASIC can potentially get bottlenecked by the CPUs!!
We can certainly do that though with a NIC, the result is called a System-on-Chip NIC:

![[Pasted image 20240424130057.png]]

They are very programmable and have ample memory (you can run Linux on it!!), but again, the performance is lacking, it just cannot go up to more than 10 Gbps. The only way to work with this and get 40 Gbps is to use **FPGAs**, and that was the choice that AccelNet folks made!

There are also other reasons that FPGA was chosen:
- The folks working on this project already had significant experience with FPGAs
- NICs don't need too complicated of a logic, FPGAs work just fine whereas ASICs can be overkill 
- It is easy to make this vendor-agnostic. You could use Verilog or System Verilog or any other VHDL language.

## Using FPGAs

Using FPGAs was a huge undertaking, and one big question was *where to put it?*.
The previous project that used FPGAs, implemented them as a separate box, connected via a network to the servers. The reason was that since it was experimental, they had to be carful and make sure that it would not bring down a server if it had a problem. 
However, this was not good for SDN offloads.

Another solution was to integrate it directly to the NIC, making it into a an absolute beast of a circuit:

![[Pasted image 20240424131229.png]]

That definitely works, but it is a huge undertaking, and it also means that all the NIC functionalities (most importantly, RDMA) need to be re-implemented and have their drivers re-written and the VMs updated!

In the end, the choice was to go somewhere in-between. A *Bump-in-the-Wire* approach, where the FPGA sits between the ToR and the NIC and essentially acts as a packet filter:

![[Pasted image 20240424131415.png]]

As for what the FPGA does, conceptually it is pretty simple:

![[Pasted image 20240424131531.png|600]]

Take the path from the NIC to the Switch for example:
1. The FPGA receives a packet from the NIC and buffers it
2. The packets are passed to a parser, that maps each flow to some identifier
3. The parser than passes the identifier to the match table, it then looks up the L1/2 caches or the DRAM if there is a miss to see what action maps to this identifier. 
4. The action micro-code is then executed on the buffered packet. This can be modifying the headers or popping labels.

You should note that the VFP software, the actual control plane software, runs on the **Host Kernel**, so it bypasses the Hypervisor.

So essentially:
- **First Packet** goes from VM $\rightarrow$ NIC $\rightarrow$ FPGA $\rightarrow$ VFPA $\rightarrow$ FPGA
- **Next Packets** go over VM $\rightarrow$ NIC $\rightarrow$ FPGA $\rightarrow$ ToR

If at any point, the FPGA needs to be upgraded or its drivers should change, the kernel will be manually notified and the traffic will go directly to the host operating system via a `VMBus` interface. The throughput drops significantly during this, but it remains transparent to the VMs. This backup path is referred to as the *Synthetic Path* at times and this process as a whole is called *Transparent Bonding*.

![[Pasted image 20240424133901.png|500]]

