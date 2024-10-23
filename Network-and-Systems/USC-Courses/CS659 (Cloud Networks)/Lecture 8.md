# Facebook BGP (cont.)

For IP assignment, we use a **single** IP prefix for each pod, and we do this because if the pod-pod network uses ECMP, it is able to use all the available paths from each pod to another.

So:
- All servers on the same Rack will have the same prefix
- All switch interface from a single prefix will have the same IP address
- The whole pod is assigned the same prefix for optimal ECMP use

For announcements: 
- **Within Pod**: Announce ***Rack and Switch*** aggregates everywhere
- **Outside Pod:** Announce **Pod and Rack** level aggregates everywhere

## Other Objectives

### Reliability

What to do if a fabric to rack switch link fails?
One way would be to wait for BGP you converge again, but that takes time. We can do better.

>[!REMINDER]- BGP Communities
>One of the important BGP policy configurations is Communities. A BGP community is 32 bit integer that is advertised alongside an announcement. It adds an additional meaning to that announcement.
>
>Two ISPs will come to terms about what each community attribute would mean, and once they receive it, the would act accordingly. For example, a tag might suggest that a route is to be dropped, or not forwarded towards a certain ASN.

We can make a very clever use of BGP communities to define backup paths:

![[Pasted image 20240214124224.png|500]]

We can pre-define backup paths using BGP communities. Just some notes before we discuss the figure:
- Usually a rack switch would not want to handle traffic of any rack other than itself, only in the event of a backup path announcement will it accept such a path.
- These backup routes are essentially precomputed using the appropriate community tags, and this allows the network to not invoke any BGP convergence at all (we remember from CS551 that BGP can take a very long time to converge)
- Backup paths never leave the pod, since the route aggregation prevents it
- Since this would go to the same Peer Group, all of this configuration is pre-computed.

We call this method **Fast Rerouting**.
For a concrete example, take the above figure:
- FSW1 is handling traffic towards RSW1.
- The link between FSW1 and RSW1 fails.
	- Now, RSW2 is **pre-configured** to add a tag to any route to RSW1 coming in from FSW1 that it is routing towards a rack prefix. 
	- It will advertise this to FSW2, and it is also **pre-configured** to match on anything tagged with being an RSW1 rack prefix to be tagged additionally as **backup**.
	- This gets forwarded to RSW1 via announcement from FSW2
	- When the direct link from RSW1 to FSW1 exists, this announcement is discarded, since RSW1 will prefer a path that does not have the backup tag
- Now, when the link dies and the non-backup route gets evicted due to timeout, RSW1 will have no choice but to accept the backup path. Thus it sends traffic towards it, and FSW2 and RSW2 will handle the rest.

Note that all of this was **pre-configure**, so **no BGP convergence needs to happen! The network has essentially encoded all possible single-link failure scenarios within itself!**

### Maintainability

Any management operation, be that:
- Rewiring of links
- Expansion of a DC
- Upgrading HW/SW

Would have two steps:
1. **Drain:** reroute production traffic away from an entity (a link, a switch, a whole DC, etc.)
2. **Undrain:** Reintroduce an entity into the network and allow it to accept traffic

These operations are done in small steps, so for example, in Jupiter:
- First we drain and remove an entity from a network
- We do the management operation (i.e. replace a switch, move some optical fiber around, etc.) **partially**, remember that we have the constrain that 75 percent of the DC should always be available, so we might have to do this in steps
- Undrain and retinroduce the device into network
- Repeat all of this until the whole operation is complete

There are 2 types of drains:
- **Soft Drain:** No production traffic is allowed, but we can still access that device. For example, when upgrading the OS of a switch
- **Hard Drain:** Completely turn a device off and do not allow it to operate in the network.

The question now is, how to even do a Soft Drain?
A conceptual way of doing this would be with a 3 state FSM. We will have 3 states for each device:
- **LIVE:** The device is actively carrying production traffic
- **WARM:** The device is waiting for the configuration that would reroute the traffic to take effect
- **DRAINED:** The configuration was ACKed and the device is no longer receiving traffic and can be safely modified.

Since BGP is handling all of this process, it is going to take a pretty long time for these changes to take effect! This is a pretty lengthy process usually, with or without BGP.

You want to be careful with this, since it is important for all of these operations to be as **Hitless** as possible; making sure that not even a single production packet of data gets dropped because of the management operation.

### Scalability

We can use BGP to summarize routes both at rack and pod level.
Also, since Facebook is using its own implementation of BGP, they can optimize policy announcement and routing much more than a commercial application of BGP.

This is where sequencing of events might also be important. For scalability, we should make sure that route policies that filter the most amount of packets are hit first. Facebook has not yet announced what their BGP implementation does, so we can only speculate.
### Service Availability 

In a DC, many services are enabled with multiple different instances (i.e. VMs). Each service can be moved and added or removed to/from any different rack server as needed.

It is important that each instances change should not require a routing re-configuration. Services are given **Virtual IPs** (VIPs) and each instance advertises the VIP to the rack switch and BGP anycast would route the VIP to the nearest instance.

Now we can handle link failures as we saw before, but what about *router failures?*. A router failure would be guaranteed to provoke a BGP re-converge, since routes from other pods could theoretically get black-holed and there is no way to encode that configuration into the pre-computed routing table without making it explode, so in order to avoid the well-documented delayed convergence problems of BGP, we need to take some measures in advance:
- Well-defined, pre-computed backup paths will be used while BGP is trying to converge.
- Prevent BGP from exploring long paths, and that can be done by limiting convergence to within a single pod (Facebook does not explain how they actually do this :/)

>[!REMINDER]- BGP Delayed Convergence Problem
>On a path of length $N$, BGP convergence *could* be delayed by $O(N!)$.


### Facebooks BGP Implementation

Facebook made an in-house BGP implementation, instead of using say, Quagga. They have noted better performance:

![[Pasted image 20240214130455.png|400]]

They list the main reasons for better performance as:
- All peers and the RIB processes are assigned to separate threads
- Their co-routine library very efficiently interleaves I/O and computation
- For installing BGP policies:
	- **Batch** import policy processing in a single operation
	- For export policies, use a **Policy Cache**, to avoid reinstalling some policies as much as possible
### Service Upgrades

When the decision is made to make any change to a service or configuration, it is rolled out incrementally in separate phases:

![[Pasted image 20240205174127.png]]

Based on the conditions, 2 mechanisms can be used to help with this process:
- **BGP Graceful Restart:** Before we disconnect a switch, we announce via BGP that we are not going to leave the network for long, and we instruct the other networks to just *deprioritize* the path to this switch instead of just dropping it. This prevents us from having to wait for another BGP route convergence after we come back. This is very good when a switch just has to undergo a configuration change, since it is non-disruptive.  This will also keep the forwarding table while the agent itself restarts.
- **Draining:** We have already seen this, it is needed for example when we do a software upgrade, but it is quite disruptive to the current service.

Some key findings have been that it takes multiple weeks for such an upgrade to finish, and even then, there will be a few devices that will NOT receive the update because they were offline during all this time.