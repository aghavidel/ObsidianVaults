# NVP (Cont.)

>[!NOTE]
>If the last stage of a Datapath pipeline is a L2 switch, then it is executed in the ingress of the destination rather than in the egress of the source.

## Forwarding State Computation

![[Pasted image 20240401160411.png|500]]

State must be calculated for every tenant. What that means, is that the inputs:
- Location of the VMs (virtual NIC assignments) and MAC addresses
- Tenant configuration information

And the outputs (the logical Datapath at each node) must be done for *every* tenant. This is a headache since it is common for tenants to slightly change the configuration repeatedly.

Since this state can be very large, we need to scale it down. To do this, we leverage incremental state computation using `nlog`, which is based on [[Datalog]], a logic programming language that supports incremental updates.

>[!EXAMPLE] A Datalog Example
>Datalog just evaluates feasible sets of first-order logic statements. So for example, if we have:
>$$
>T := (X > 0) \wedge (Y > 5)
>$$
>Then Datalog will just output a sample in $\{X > 0, Y > 5\}$, such as $X=1, Y=6$. Now, if the second conjunction is changed to say $Y > 6$, then Datalog realizes that it only needs to make $Y > 6$ feasible, since $X > 0$ is still true from its previous evaluation.

For a more realistic example:

![[Pasted image 20240401161547.png|600]]

Now:
- In above, each comma separates a statement with a conjunct (i.e. just replace the commas with $\wedge$). The `tunnel` statement evaluates to true, if and only if all its sub-statements (those being `log_port`, `log_datapath_encap`, etc.) are true.
- The output of Datalog would be a choice of the arguments in `tunnel` (i.e. source and destination logical port, tunneling and destination IP) that makes sure `tunnel` evaluates to true.
- As for what the tunnel statement actually means:
	- Pick some logical port on the source and destination port and choose a tunnel (that is what `encap` means)
	- See where these logical ports are attached to
	- Map the destination IP address to the chosen VM
	- Make sure that the source and destination VMs are not equal (why use a tunnel when destination is local to the source?)

Now, assume a VM migrates to a new machine, what should we do?
If we have $N$ VMs, then we have at most $N(N-1)$ tunnels in total, however, we only need to modify $N-1$ tunnels at most. Datalog notices this and this is what incremental update really means.

## Scaling The Computation

![[Pasted image 20240401163443.png|500]]

The actual computation can be done in parallel. However, to fully utilize incremental updates, we use a **Logical Controller** that generates an abstract Datapath for each tenant *only once*.
The benefit of this intermediate representation is that it allows for the users to change things very easily (so if a user does not like an IP allocated by the system, we just assign it directly in the logical representation).

Anything that was left from this process will be filled by the physical controllers with real data and then pushed into the network with OpenFlow.

NVP has been very successful, and a few important reasons for that are:
1. It allowed importing of tenant configuration without any modification (that is the power of virtualization)
2. `nlog` allowed correctness checking and incremental updates (easy to configure and fast)
3. Use of OVS ensured network independence

# Andromeda 

Andromeda is Google's cloud virtualization software. They want:
- Network isolation based on customer policies
- Management features (billing, firewall, etc.)
- High availability and support for live migration
- Scale to hundreds of thousands of VMs
- Support customers spanning multiple clusters

Andromeda has some important differences compared to NVP:
- Andromeda is not fully proactive like NVP, this allows it to scale better. NVP computes logical Datapath for every pair of connected VMs
- Andromeda supports *Fast Path Processing*, which means that it tries very hard to have a change of networking intent take effect in less than some time (maybe less than 1 milli-second).

![[Pasted image 20240401165314.png|500]]

Here:
- **Fast Path:** Quickly process packets from *Active Flows*
- **Co-Processor:** Slower, heavier processing that should not be done on the fast path (e.g. encryption)
- **Hoverboard:** Process small, rarely active flows

Using an entity-relationship model (essentially like the one used in Onix), we can:
- Represent network objects as entities (VMs, for example)
- Set properties (like IP addresses)
- Relationships (like which VM will run which host)

And that relation can be exposed to create intents. All of this intent is given to the VM controller frontend like the following:

![[Pasted image 20240401170606.png|300]]

The OpenFlow Frontend is just another name for the OpenFlow Controller. It will report events like a VM going down or joining a network.

For each host, we run Open vSwitch like the following:

![[Pasted image 20240401170815.png|400]]

The data plane will have its own dedicated cores on the host machine to be able to cope with high utilization from the VM.

## Main Idea

Now we come to the main idea of Andromeda. We have seen **proactive** network programming as in NVP:
- Insert all flows from intent at once
- My not scale well, since it is $O(N^2)$, but you only need to do it once and the rest can be incremental

With a Reactive programming:
- Insert flow entries when traffic starts
- Can cause a delay, but at least it scales better

The key idea is to give only very few, very large flows to the hosts. Give the rest to the Hoverboard switch, a software switch that can support a large flow table. 

So it kind of looks like the following:
- If a VM A has no way to send a flow to VM B, then it sends the traffic to the hoverboard
- If hoverboard notices a large flow, then it reaches out to the VM controller, and it will then get instructions from the controller to program VM A to handle sending packets by itself.

What is the point of this?
- If you have a small traffic, then Hoverboard will handle it by itself, the VMs (and by extension their associated Datapaths) need not be changed.
- If there is a large traffic that can potentially overwhelm the hoverboard, it is offloaded to the  hosts themselves, and that requires modification to Datapaths. 
- What makes this scalable is that the modification need only happen rarely (we probably are not expecting to see many large flows, and that is pretty consistent with observations in the DCN)

This however, only works if the following conditions are met:
- Not all pairs communicate all the times (so communication patterns are pretty sparse)
- Most flows are small

![[Pasted image 20240401173351.png|500]]


## Live Migration

Migrating a VM means:
- You have *live* VM that has real connection to some destinations and some volatile data in it
- You want to pick it up and put it in another machine without causing it to break

The simplest way is to:
- Pause the VM (hypervisors let you do that)
- Copy the VM over the network and send it to the destination
- Start the new VM
- Delete the old VM

However, this process needs to be fast, since while the VM is paused, the customer will see a downtime. To minimize this, we need to make sure that we can handle large or lengthy transfers quickly, but more importantly, we need to make sure we don't break TCP connections.

For transferring, we need to defer pausing the VM, and so we need to slowly copy the VM bit by bit while keeping it consistent and only pause the VM for a very short amount of time.

The main idea is this:
- The new VM needs to have the same IP as the original, and this can be done by tunneling.
- However, when a VM is paused, it is unable to answer packets received from other VMs. TCP can timeout and people will be angry.
  The solution is very clever. When the time of the original VM pause comes, the VM controller will install a **hairpin entry** on the original VM.
  A hairpin entry will reflect any packet came to it and send it to the destination of the new VM. Since the new VM is not yet operational, these packets will be buffered in the destination.
- Now, Andromeda will wait for the hairpin to take effect. The moment that happens, the VM will no longer receive anything, and  that instant the system will pause the VM, transfer the last remaining part of the VM, and start the new VM.
- The new VM wakes up and serves the buffered packet, preventing TCP timeouts on people who sent to the original VM.
- The original VM itself is deleted and all of its resources are finally relinquished, the migration is now complete.

All of this process should take **a few microseconds**!!
