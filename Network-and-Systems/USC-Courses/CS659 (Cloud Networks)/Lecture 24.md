# 1RMA (Cont.)

## RDMA Challenges

RDMA is very much implemented purely in hardware, and it is also connection oriented, and that needs connection state in the hardware, which requires memory on NIC, which is small. Implementing everything in hardware makes it hard to evolve.

RDMA also was too general for Google's needs. They had to make it into a more refined and specialized construct. 

To address these:
1. Use one-sided RDMA
2. Split the implementation both into hardware and software

As such, 1RMA has 3 important properties:
1. It can be connectionless, which makes the NIC much simpler
2. Completely one-sided
3. Implements congestion control in software (so this part needs the CPU)

We also saw examples of how RDMA works, for example with read:

![[Pasted image 20240415161037.png]]

- Steps 1, 2 and 3 (i.e. registering memory regions and adding operation to NIC queue) needs the CPU
- The rest are done between NICs

## Using Fixed NIC Resources

![[Pasted image 20240415162348.png]]

At any point of time, the host maintains a *Solicitation Window*, which is the window allocated to inbound transfers and is scaled based on the BDP. 

It is important to note that each step numbered above can fail. One question is how do we handle them exactly? For example, slot table can be full, or operations may timeout. The 1RMA NIC should have hardware in place that can notify the software. This is necessary since we do not have the kernel to handle interrupts for us. 

For sending operations, the steps that happen are the following (this is essentially what happens in step 4 of the previous lecture):
1. Host writes an operation into the command slot table (CST)
2. Queue Arbiter processes will then pick from that table and transfer them to the outgoing operation queues when the solicitation window has space for it.
   Scaling the window based on BDP is where congestion control is done.
3. If there is space, the operation is admitted, if not, it is kept in memory as CST entry until we can admit it.
4. Operations are then encrypted and passed to the AES blocks

So operations that are kept in the table for too long can actually timeout.

From the application point of view:

![[Pasted image 20240424151830.png]]

Things are simpler, the applications communicate with a `CommandExecutor` that chunks large operations and provide pacing and congestion control.
The backend is managed by a `CommandPortal` which manages the CST and hardware resources. It also reuses CST slots when an operation gets completed.

# Maglev

This paper discusses the implementation of a fast and reliable software network load balancer. 

## Background: Load Balancers

The way all web services work usually is that a client sends a request and establishes a TCP connection, and then uses HTTP/HTTPS to send actual data. Many clouds also use a REST API for this which does the same. The problem here however is a problem of scale.

We know that we have to scale out, and we have seen that a lot, but the problem is how to map clients to instances, and how to dynamically manage that when the size of the server pool changes?
The device that does this is called a *Load Balancer*.

![[Pasted image 20240415170300.png|400]]

A few decades ago, load balancers were actually hardware boxes. The typical LB needs to have some properties:
- **Liveness**, which means that it should always know which server is up so that it can serve requests to a working server.
- **Persistence**, serve multiple requests from the same client to the same server.

The LB has some good services it can provide:
1. **Scale Out** applications and services based on client demand
2. **Resource Sharing**, same server can run different instances
3. Simplify maintenance 

How we do load balancing per-se is not really that important. What is important though is what is called *Service/Server Association*, which basically means how the LB keeps data that allows it to map a client/service to a particular server. This data is crucial in order to provide persistence. 

There are many ways to do this, usually though it boils down to using either the IP header and TCP header info, or higher level data (like cookies, URLs, and many more). Keep this in mind, it'll come back.

## Maglev: Main Idea

The main idea of Maglev is to scale out the LB itself by having many software LBs. Put a router in front of the LBs to route traffic. No need to be worried about this, since we know how to make really big and scalable routers with Merchant Silicon.

![[Pasted image 20240415172051.png|300]]

Note that a tenant can use some of their own VMs as LB as well. 
The way this integrates into the web ecosystem is like the following:

![[Pasted image 20240415172753.png|500]]

Basically:
1. A client reaches for `google.com`, the DNS resolves this to some front-end near the client (this is very similar to how the Akamai CDN works).
2. It goes through the WAN and reaches the router of the front-end. These things are called literarily Google Front-Ends or GFEs. These really are not routers, but actual servers.
3. GFEs talk with BGP with Maglev instances, and each Maglev instance stays one hop away from the router and announces a Virtual IP address to the GFE.
4. The GFE does ECMP on its outbound towards Maglev instances, so there is one level of balancing there which is TCP friendly (keeps the sessions as long as no one dies).
5. When a Maglev instance receives a packet, it checks the service endpoints for which one can be used and sends the request to them (this is step 2)
6. Finally, the response does NOT go through Maglev instances, since responses can be very large, they go directly to the front-end and routed towards the WAN.

Each Maglev box is fairly simple:

![[Pasted image 20240415173459.png|300]]

There are two parts, the controller and the forwarder. 
- The Controller, just makes sure the forwarder is up and running announces or withdraws Virtual IP addresses if needed. 
- The forwarder keeps the data about what endpoint is up and which backend is available.
- There is also a Configuration Manager that specifies the set of VIPs and BPs for the Maglev instance and can update each instance atomically.

