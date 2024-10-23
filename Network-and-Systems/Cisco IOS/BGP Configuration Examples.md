# BGP Configuration Case Studies

This note picks of from our [[BGP]] note and offers case studies on BGP configuration within the framework of Cisco IOS XR routers.

## EBGP Peering

The simplest of all BGP configuration, routers R1 and R2 with ASNs 100 and 200 are connected via a link between `192.168.1.225/24` and  `192.168.1.226/24`. 

Note that these are point-to-point connections, meaning that the interfaces themselves are not necessarily directly connected (or even real physical interfaces), these can be connected with an underlying network of IGPs for instance.

They are peered in an EBGP session like the following:

```
======== R1 ========
router bgp 100
	neighbor 192.168.1.226 remote-as 200
======== R2 ========
router bgp 200
	neighbor 192.168.1.225 remote-as 100
```

Once this configuration is pushed, we can check for what each router is doing using `debug ip bgp`. For example, running on R1 we'll get:

```
BGP debugging is on for address family: IPv4 Unicast 
R1# 
BGP: 192.168.1.226 open active, local address 192.168.1.225
BGP: 192.168.1.226 open failed: Connection refused by remote host
BGP: 192.168.1.226 open active, local address 192.168.1.225 
BGP: 192.168.1.226 went from Active to OpenSent 
BGP: 192.168.1.226 sending OPEN, version 4, my as: 100, holdtime 180 seconds 
BGP: 192.168.1.226 send message type 1, length (incl. header) 45 
BGP: 192.168.1.226 rcv message type 1, length (excl. header) 26 
BGP: 192.168.1.226 rcv OPEN, version 4, holdtime 180 seconds 
BGP: 192.168.1.226 rcv OPEN w/ OPTION parameter len: 16 
BGP: 192.168.1.226 rcvd OPEN w/ optional parameter type 2 (Capability) len 6 
BGP: 192.168.1.226 OPEN has CAPABILITY code: 1, length 4 
BGP: 192.168.1.226 OPEN has MP_EXT CAP for afi/safi: 1/1 
BGP: 192.168.1.226 rcvd OPEN w/ optional parameter type 2 (Capability) len 2 
BGP: 192.168.1.226 OPEN has CAPABILITY code: 128, length 0 
BGP: 192.168.1.226 OPEN has ROUTE-REFRESH capability(old) for all address-families 
BGP: 192.168.1.226 rcvd OPEN w/ optional parameter type 2 (Capability) len 2 
BGP: 192.168.1.226 OPEN has CAPABILITY code: 2, length 0
BGP: 192.168.1.226 OPEN has ROUTE-REFRESH capability(new) for all address-families 
BGP: 192.168.1.226 rcvd OPEN w/ remote AS 200 
BGP: 192.168.1.226 went from OpenSent to OpenConfirm 
BGP: 192.168.1.226 went from OpenConfirm to Established
```

At first, BGP fails to be established, since R2 was yet configured for it at the time, with the next attempt, R1 manages to establish a TCP connection with R2 and send an `OPEN` message to it. With this, it changes it's state to `OpenSent`.

Awaiting response from R2, it then receives a message from it that negotiates parameters an capabilities. Once accepted, R2 transitions to `OpenConfirm` and then to `Established`.

Once the connection is established, an entry is added into the IOS local database for BGP neighbors. These can be inspected with `show ip bgp neighbors`:

```
BGP neighbor is 192.168.1.226, remote AS 200, external link 
	BGP version 4, remote router ID 192.168.1.226 
	BGP state = Established, up for 00:23:25 
	Last read 00:00:25, last write 00:00:25, hold time is 180, keepalive interval 
	is 60 seconds
...
...
Connection state is ESTAB, I/O status: 1, unread input bytes: 0 
Connection is ECN Disabled, Mininum incoming TTL 0, Outgoing TTL 1 
Local host: 192.168.1.225, Local port: 13828 
Foreign host: 192.168.1.226, Foreign port: 179
```

We excluded some details, but this can be interpreted as:
- We have settled on BGPv4
- BGP is currently in the `Establsihed` state
- The router ID of our neighbor is `192.168.1.226` and resides in AS number 200
- Connection is established on R1's port 13828 and R2's port 179 (as expected)


## IBGP Peering
Here is an example:

![[Pasted image 20221013012424.png|500]]

Recall that a direct IBGP session between any BGP router within an AS is required.

To implement this topology, we'll do the following:

```
======== Vail ========
router bgp 100 
	neighbor 192.168.1.197 remote-as 100 
	neighbor 192.168.1.222 remote-as 100 
	neighbor 192.168.1.225 remote-as 200
======== Aspen ========
router bgp 100 
	neighbor 192.168.1.197 remote-as 100 
	neighbor 192.168.1.221 remote-as 100
======== Telluride ========
router bgp 100 
	neighbor 192.168.1.198 remote-as 100 
	neighbor 192.168.1.205 remote-as 400 
	neighbor 192.168.1.221 remote-as 100
```

We can now check for a summary of the state of a router in the BGP session using `show ip bgp summary`:

```
Aspen#show ip bgp summary 
BGP router identifier 192.168.1.222, local AS number 100 
BGP table version is 1, main routing table version 1 
Neighbor        V   AS   MsgRcvd  MsgSent  TblVer  InQ  OutQ Up/Down   State/PfxRcd 
192.168.1.197   4   100  12       20       1       0    0    00:15:43  0 
192.168.1.221   4   100  23       30       1       0    0    00:26:14  0
```

In this example, unlike the previous one, an IGP is actually *necessary*. Without an IGP, the session between Vail and Telluride cannot be established, since neither would know how to reach the others interface (note that they are not connected directly, they rely on Aspen to connect them).

>[!QUESTION]
> Say we add a link between Telluride and Vail. Doing so means that a link failure in AS 100 will not disable connections between AS 200 and 400. Like below:
> 
> 
> ![[Pasted image 20221013015305.png|500]]
> 
> 
> But how would you route the IBGP sessions? If the link is gone, the physical interfaces go with it, and so you would need to actually add a new configuration to the router!
> Can you think of some way to prevent this?

>[!ANSWER]-
>Use **Loopback** interfaces for BGP router IDs!
>Loopbacks always have a route on the router (i.e. a route that says that this destination is reachable via any physical interface on the router). 
>
>Using these interfaces instead of the physical interfaces on the router, ensures that even if physical links die, the IGP will still route packets on the best path possible, and no reconfiguration of BGP would be required!
>
>You might ask about what if the loopback itself dies? Well, as long as a router lives, the loopback also lives, since it is a software implementation on the router, not an actual physical interface.

Assuming you've read the answer above (or already guessed it), we first enable some loopbacks on each router like the following: 

![[Pasted image 20221013015951.png|500]]


You can modify the code we wrote like the following:

```
======== Vail ========
router bgp 100 
	neighbor 192.168.1.197 remote-as 100 
	neighbor 192.168.1.222 remote-as 100 
	neighbor 192.168.1.225 remote-as 200
======== Aspen ========
router bgp 100 
	neighbor 192.168.1.197 remote-as 100 
	neighbor 192.168.1.221 remote-as 100
======== Telluride ========
router bgp 100 
	neighbor 192.168.1.198 remote-as 100 
	neighbor 192.168.1.205 remote-as 400 
	neighbor 192.168.1.221 remote-as 100
```

But this by itself is not enough. By default, the "source" label of an outgoing TCP session is set to be the address of the outgoing interface. Loopback interfaces are NOT the outgoing interface, some physical interface of the router will be that source instead. We need to specifically state that we want the source to be the loopback address instead.

To this end, the `neighbor update-source` directive is introduced, which accepts the name of an interface that it will then use for it's source address. Modifying the snippet above will now yield:

```
======== Vail ========
router bgp 100 
	neighbor 192.168.100.2 remote-as 100
	neighbor 192.168.100.2 update-source Loopback0 
	neighbor 192.168.100.3 remote-as 100 
	neighbor 192.168.1.225 remote-as 200
======== Aspen ========
router bgp 100 
	neighbor 192.168.100.1 remote-as 100
	neighbor 192.168.100.1 update-source Loopback0
	neighbor 192.168.100.3 remote-as 100
	neighbor 192.168.100.3 update-source Loopback0
======== Telluride ========
router bgp 100 
	neighbor 192.168.100.2 remote-as 100 
	neighbor 192.168.100.2 update-source Loopback0
	neighbor 192.168.1.205 remote-as 400 
	neighbor 192.168.100.3 remote-as 100
	neighbor 192.168.100.3 update-source Loopback0

```

Of course, care must be taken to make sure that the loopback addresses are actually advertised with an IBGP.

## More EBGP Peering

For now, our examples were simple. Now we go on doing more complicated examples. Before we proceed, let's state two important caveats to the IOS implementation of EBGP.

>[!IMPORTANT] Important Caveats For EBGP In IOS
>EBGP sessions are subject to the following conditions by default:
>
>- It is assumed that the EBGP neighbor exists in a reachable subnet of the current router and is directly reachable. Neighbors that are not directly connected will not be accepted by default.
>- EBGP messages by default have a TTL of 1.
> 
>These limitations must be changed with a manual override by the user if necessary.

Let's show an example. Imagine we wanted to use the same loopback address technique that we discussed above with EBGP. As you probably have guessed, it won't work immediately!

Let's implement the peering for the following:

![[Pasted image 20221021014342.png|500]]

Of course, a static route is also needed that says that "the loopback address of the next router is reachable via it's physical interface" for each router. This alone means that the endpoints are NOT directly connected and hence any moment that the IOS attempts to establish connection, it will fail immediately because of the caveats that we mentioned above.

Solution? Just override the default configuration! 

IOS provides the `disable-connected-check` directive for this. So we'll have:

```
======== Alta ========
neighbor 192.168.200.3 remote-as 100
neighbor 192.168.200.3 update-source Loopback0
neighbor 192.168.200.3 disable-connected-check
```

There is also the need to establish BGP sessions in completely point-to-point manner, meaning that the BGP session must traverse over networks of routers that may not be running BGP at all!

Normal setup on these networks also fails, not only due to the first default config, but also the second one, since it also forces the TTL to be just 1, hence the BGP message will be dropped as soon as it reaches a new router other than the destination.

Here is an example of this kind of configuration:

![[Pasted image 20221021015430.png|400]]

Simple IGP session would route traffic between Telluride and Copper. A static route needs to be installed on Copper and Alta to create a path between them. So using simple [[OSPF]] for example, we'll have the following:

```
======== Telluride ========
router ospf 1
	network 0.0.0.0 255.255.255.255 area 0
!
======== Copper ========
router ospf 1
	redistribute static 
	network 0.0.0.0 255.255.255.255 area 0
!
ip route 192.168.200.1 255.255.255.255 192.168.1.222
======== Alta ========
ip route 192.168.100.3 255.255.255.255 192.168.1.221
```

So now for BGP. As we said, Copper will NOT run any BGP. So we only need to configure the remaining two routers. To do this, we also need to override both of the default BGP configs that we mentioned above. We already see how to that with the former, let's see how you would do it with the latter.

To override the default TTL, the `ebgp-multihop` directive is used. This directive also accepts a new value for the TTL. Since we need at least 2, let's put it at that.

```
======== Telluride ========
router bgp 100
	no synchronization
	bgp log-neighbor-changes
	neighbor 192.168.200.1 remote-as 400
	neighbor 192.168.200.1 ebgp-multihop 2
	neighbor 192.168.200.1 update-source Loopback0
!
======== Alta ========
router bgp 400
	no synchronization
	bgp log-neighbor-changes
	neighbor 192.168.200.3 remote-as 100
	neighbor 192.168.200.3 ebgp-multihop 2
	neighbor 192.168.200.3 update-source Loopback0
!
```

>[!FAQ] Why Haven't We Wrote The `disable-connected-check` Directive?
>By default, once `ebgp-multihop` takes effect, `disable-connected-check` also takes effect, since it wouldn't make much sense to leave it on.

>[!REMINDER] 
>Remember that TTL is decreased if and only if we leave from the physical interface of a router (i.e. a *true* hop). Hops between loopbacks and physical interfaces do not count, they aren't really hops!


### Some Security Considerations

BGP as a whole is *really really* hard to make safe in large networks. Even now, attacks can successfully bring down BGP peering between multiple routers and setback network function for entire hours until the network recovers the sessions. So you should consider making things safer, especially if you wish to deploy networks in the wild and aren't just using lab equipment.

We'll briefly go over things that you can consider.

#### The `router-id` Directive

IOS needs to put some source address on the BGP messages it sends, and since the `update-source` isn't necessarily always active, it has a default procedure where it uses the highest IP address of all the available loopback addresses for the source IP. If none are found, it will fall back on doing the same for physical interfaces.

The `router-id` directive that we mentioned overrides this procedure, and makes sure that packets are sourced from where we actually think they are sourced, it is very useful to have this in configurations.

#### Log BGP Activity

New IOS releases always have the `bgp log-neighbor-changes` directive enabled by default, meaning that using the command `show logging`, we can see what each neighbor is doing and what state each of their interfaces are.

You can also configure Syslog if you wanted to.

#### Use MD5 Authentication

MD5 authentication can be enabled, allowing for each router to have a password, and each peer wanting to execute a `neighbor` directive with that router, must provide this password, and vice versa.

>[!IMPORTANT] Considerations For Passwords
>While you might be able to get away with IBGP sessions not having passwords, EBGP sessions MUST have a password, not only that, it should preferably be unique to that peering and only that peering alone. 
>EBGP is the least safe part of the BGP specification (understandably, since it is the only part that truly "leaves the nest" and goes into another domain), so you should try to make them as safe as possible.
>IBGP sessions are also advised to have at least some password (even if it is shared among all of them).


#### Use 




