# Network Virtualization in DCNs (cont.)

>[!NOTE] 
>Usually, the word "*Tenant*" refers to the clients of a cloud network. The reason that multi-tenant is in the name of the paper, is because most VM technology serves only a single client (at least at the time!)

>[!NOTE] Defining "Isolation"
>When we say that VMs should be isolated, we mean that in two ways:
>- **Security Isolation**: The two VMs cannot exchange anything with each other without a 3rd party.
>- **Performance Isolation:** No matter what a VM does, it cannot impact the performance of another VM.

## Basic End-Host Virtualization

The most basic kind of network virtualization is to hide hosts on endpoints. That is easily achieved by using tunneling techniques. We have seen a bunch of them so for now, we just use IP-in-IP encapsulation:

![[Pasted image 20240325172204.png|300]]

In the above example, the green headers contain the IPs of virtual hosts X and Y, while the red header which *encapsulates* the green header, contains the IP addresses of A and B, the actual physical nodes.

The blue network is usually referred to as an **Overlay Network**.

So this is how virtualization is done by hosts, but how about using the network directly?

## Physical Network Support For Virtualization

The most basic version of this would be of course VLANs, where the packet would include some data in its header that identifies which overlay network it belongs to. Switches will then build a table specifically for each overlay network and use those to route packets. Having different tables for different overlays ensures **isolation**.

>[!FAQ]- Why Even Use This Instead of Encapsulation?
>Encapsulation requires cooperation from the end hosts. That is too much to expect in some systems. Say right here in USC! If someone wants internet access from a socket on the wall, they just plug in an Ethernet cable and they just want internet directly.
>
>VLANs are good for that, since all of the tagging is done by using the switch/access point that directly serves that port, NOT the host!

Usually, the most basic support for virtualization would be:
- VLANS for L2 MAC lookup
- VRF for IP lookup (VRFs are tags that go directly into an unused portion of the IPv4 header)
### Background: Virtual Switches

To manage virtual networks and hosts, VMs will make use of a piece of software that handles all of this, and one of these software is called a *Virtual Switch*. The most famous one that we know and love is Open vSwitch which we have plenty of experience with.

![[Pasted image 20240325173644.png|500]]

The OVS daemon and its kernel Datapath that does the actual packet handling will be located in the VM and does the actual routing and switching. Of course, this means that at least some part of the VM CPU needs to be dedicated to this and this is why the kernel Datapath needs to be very efficient (this creature is called DPDK in Linux).

The `ovs-vswitchd` dynamically determines what to do with each packet that the kernel Datapath does not know how to handle. It either does this on its own or receives appropriate commands from the controller via OpenFlow.

The controller receives data from the `ovsdb-server` for deciding what to do with all requests received from `ovs-vswitchd`. This is quite similar to reactive OpenFlow controllers as we have seen before, though  there is no hardware switch involved in this at all.
The controller sits outside of the VM, only the switch daemon and the DB server sits in the VM box.

>[!IMPORTANT]
>All OVS actions (i.e. all `ovs-vswitcd` flow matches) are done with **Exact Matches**, and that is one of the main reasons that the kernel Datapath and the switch daemon are separated in the first place. The default Linux routing stack does not provide anything efficient for exact matches, so the kernel Datapath does it instead.

### Virtualization Motivation

There are many reasons that virtualization was very nice for clouds. Most important being that without virtualization, it is very hard to:
- Move VMs between physical servers
- Allow independent IP addresses as customers want
- Allow co-existence of IPv4 and IPv6

However, one of the bigger reasons, and a more implicit one, was **cloud providers want to impose policies on a packet level for each VM**.