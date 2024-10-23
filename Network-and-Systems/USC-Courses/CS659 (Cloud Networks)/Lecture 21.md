# Andromeda (Cont.)

We finished the previous lecture by discussing migration a bit. Now, we make it a bit more concrete by discussing the approach in Andromeda.

## VM Migration

![[Pasted image 20240403161851.png|600]]

- Before the migration, there is a Brownout phase, where the cluster starts to transfer the VM state to a provisioned target VM.
- At the end of the Brownout phase, the hairpin flows are installed, and the moment they take effect, the source VM is paused.
- Pausing the VM initiates the Blackout phase, and during this time, the system aggressively copies the remaining state of the source to the destination.
- Once copying is finished, the target VM is started and the source VM is gracefully stopped.

## Scaling

![[Pasted image 20240403162354.png|600]]

- The Cluster Manager (CM) is a frontend that interfaces with all the available Virtual Machine Controllers (VMCs).
- VMCs are replicated, each one running 3 instances (both to tolerate a single failure and also be able to do voting).
  The system uses a distributed locking system for leader election and voting. Google uses its own implementation, *Chubby*, instead of Zookeeper.
- The system interfaces with multiple OpenFlow Frontends (OFEs) for scalability an also works with multiple hosts. Usually, a VMC manages a single host.
- Only the associated VMC will serve the events generated from its host VM. The only event that gets broadcast across the entire cluster, is **VM startups**.

>[!IMPORTANT] Why This Many VMCs?
>Large customers can delay smaller ones. Imagine a queue of work requests, operations that you want to program into a network.
>Now, imagine we give 2 hosts to a VMC. The VMC should maintain 2 event queues for these hosts. Now, how to serve them?
>
>The thing that you DO NOT do is have a single queue. If you have a single queue with simple FIFO policies, then if a customer produces a huge amount of operations, the operations of the other client will get delayed (this is called **Head-of-Line Blocking**). We need a Fair Queuing policy to help this. 
>The simplest way of implementing fair queuing is to just use Round-Robin, but even that won't work if the size of jobs are very different.
>
>So fair queuing helps **Isolate** clients and hosts, and to achieve that, we need multiple VMCs.

If during any point of operation, a host notes that the VMC has failed, then it will **Fail Static**, it will use the last known updated state from the VMC until the connection is resumed.

## Dataplane 

![[Pasted image 20240403164145.png]]

There is quite a bit to unpack in this figure.
Each VM maintains transmission/receive queues in their virtual NICs. These queues are just producer-consumer pairs as software.

The Dataplane lives on top of the hypervisor and OVS `vswitchd`, which handles packets that cannot be handled locally (this is what we call the **Miss Path**). There is a also the **Fast Path**, which maintains a high privilege process in the data plane that is able to actually access the physical NICs directly. 
This can be used to handle latency sensitive packet transports by bypassing the kernel (i.e. it is doing RDMA).

Note that none of this is done in the application level, all of it is done in packet level, and the decision on what should and shouldn't go on the Fast Path is done solely by the cluster managers.

The Fast Path can do significant packet processing if needed:

![[Pasted image 20240403170332.png|600]]

Each box above is a **Network Function**, some process that you want to do on a packet, and this whole chain is called a **Network Function Chain**.
- We start from the top box, where flow misses, decryption, VM migrations and other expensive operations are given to the data plane, and when that is given to the flow table, it can decide what should go on the fast path and what should not.
- The actual fast path starts from the second row (labeled `(1)`). This chain will:
	- Put the job on per queue VMs
	- Give it to a priority scheduler, this is where some VMs are given more priority than others, depending on how much money they paid!
	- After that, the rest of the chain will do the rest of the offloading and then give the data to the receiving queue.

There are two interesting tricks to mention:
- **Flow Caching:** When you have a lot of flows, looking them up becomes slow, so one way to optimize this would be to have a flow table cache, indexed with a hash, and then after looking up a flow, cache the hash of that flow and that flow itself in the table and look that up next.
- TCP dump is done to get the packet data efficiently and then lookup in software:

![[Pasted image 20240403172131.png|400]]

Having these function chains is of course complicated, but it is also very configurable and can be used to create a lot of varied functions.
Contrast this with NVP, where the functions were pretty limited, they were just OVS table functions with a different coat of paint. 
It is for this reason that Network Function Chaining is now pretty much an industry standard practice.

# Accelerated Networking

**Paper:** Azure Accelerated Networking: SmartNICs in the Public Cloud

## Background: NICs

![[Pasted image 20240403173341.png|500]]

NICs expose *Ring Buffers* that are in memory, setting across the bus, and the OS is able to DMA data onto these buffers so the NIC can send them.
These buffers will contain packet descriptors that point to the actual packet in the memory of the kernel. 
Why do this?
- NIC memory is expensive, so it cannot be large, so it will only point to the packet living in the RAM instead of keeping the whole packet.
- It is also important to note that when the application wants to send a packet, they cannot pass the packet buffer directly to the kernel, since that can violate the integrity of the kernel (remember, the kernel can never trust the correctness of the user level applications).

Things become much more sluggish with VMs:

![[Pasted image 20240403173743.png|400]]

Here, when a VM wants to access the NIC, 2 copies need to happen instead of 1:
- Once from the VM to the hypervisor
- Once from the hypervisor to the kernel (though this can be optimized)

### Optimization: Single-Root IO Virtualization (SRIOV)

This is one of the first significant and practical kernel bypass mechanisms without too much hardware support.
Here, the VMs maintain access to virtual NICs, which are just pieces of software *on the NIC*, and then the NIC will provide a NIC switch, that constantly polls the virtual NICs for data.

>[!FAQ] Why Polling?
>There is no kernel here to help us, so there are no drivers (beyond the vNIC interfaces) and so **there are no interrupts!** So we have to poll the queues in the NIC to do this.
>Luckily, the NIC will do this for us (in Andromeda for example, we have the same problem, and in there, a thread was dedicated to doing this!)

Using SRIOV will completely bypass the kernel and the hypervisor from the VMs to the vNIC, and the only guarantees that we need are:
- The NIC being able to handle these
- Each vNIC is attached only to a single VM (though each VM might have multiple vNICs)

Pretty cool right?

### SRIOV In Clouds

In clouds, SRIOV does not work.
Tenants require per packet processing, and these are capabilities that can only be done using virtual switches (so like OVS), which **live in the kernel!**.

Using SRIOV will mean that the virtual switches will not see these packets at all, and that is just not good at all.

In NVP and Andromeda, they just flat out **refused to use SRIOV**, instead, they implemented a virtual SRIOV interface in the user space as part of their Dataplane.

One solution of course would be to do the packet processing in the VM, but that is not a good policy, costumers want clean abstractions, they do not want the cloud provider to tell them to do something in their own VMs.

The solution? Give it to the NIC!
- The NIC should be programmed to do this (i.e. with an ASIC, an FPGA or a CPU)
- This will free the VM CPUs from the burden of processing packets
- It is also nice for the cloud providers! More CPUs that are free, is more CPUs that can be given to a customer to pay for!