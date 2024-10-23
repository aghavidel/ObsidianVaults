# Intro
BGP stands for *Border Gateway Protocol*.

It is a routing protocol, used for routing packets between large, independent, and separately administrated bodies of routers in the internet. Some people have called BGP the *Postman* of the internet to give a bit more of an intuition about how it works.

BGP was designed to be:
- **Scalable**
- **Fully Controllable**
- **Backward Compatible**
- **Keep routers mutually suspicious**

BGP was designed to overcome the limitations of its predecessor [[EGP]] and the problems that prevented [[ARPANET]] from scaling more.

While EGP was merely a neighbor reachability protocol, BGP is a full routing protocol, that can do everything that EGP did, with even more versatility.


## Fundamentals

### Definitions
For sake of consistency, let's define a few buzzwords:

- **Autonomous System:** Or "AS" as we will call it from now on, are a set of routers, separated from the outside domain
  by a set of "edge" routers. These systems are typically under the administration of a single entity. 
  
  Previously, ASs were defined as a domain that speaks a certain IGP and uses a consistent metric for route selection; these days however, many ISPs use multiple IGPs and dozens of metrics within their own network for service differentiation and other needs, so we won't use this definition. 
  
  ASs are numbered with a number that distinguishes them from others in the network. This number is fittingly called the **Autonomous System Number** or **ASN**.
  
- **NLRI:** Stands for Network Layer Reachability Information. It is at it's core, a set of `(length, prefix)` tuples that are advertised for a single address family, showing a destination prefix for the family. 
  
  For example, an "IPv4 Unicast" family might attempt to advertise a destination of `81.31.168.208/28` into the network; in that case, the NLRI will be `(81.31.168.208, 28)`. There are many more address families and so these tuples take many forms, but serve the exact same purpose regardless. 
  
- **Route:** Our main unit of information, a pair of the form `(NLRI, path-attributes)`. Semantically, this means that the set of destinations that are aggregated in the `NLRI` portion, have a path with attributes given in the `path-attribute` portion. The latter takes many forms and might contain many fields not shared between other classes.

- **BGP Speaker:** Or just *Speaker* most of the time, is simply just the fancy name of a router that implements the BGP protocol. Basically, these consist of edge routers and internal peers of these routers.

- **BGP Peers:** What makes BGP scalable and much less chatty compared to IGPs, is that it only establishes a connection with a select few speakers, in or outside its own domain. These are called the *Peers* of that speaker. 
  
  They may be located inside the network of the speaker (in which case they are fittingly called *internal peers*) or outside the original network, making them *external peers*. BGP distinguishes these two peers and serves them differently as we shall see. 
  
  The protocols that govern the connection internally and externally are (you guessed it) Internal BGP and External BGP, abbreviated of course as **IBGP** and **EBGP**. As to why this distinction is necessary or helpful, we will see soon.


### The Issue of Trust
One of the main things that motivated the creation of BGP (and EGP before it) was that exterior domains cannot be trusted most of the time. [[IGP]] protocols assume that their domain can be trusted, meaning that:

- Routers are trusted not to be malicious
- Routers are trusted to be correctly configured
- Routers do not provide bad routing information

While IGPs can get away with this simply because the routers that they connect are under the same administration, or are completely private, BGP cannot do the same, neighbors are for the most part thieves and bandits as far as BGP is concerned.


### Should BGP Always Be Used

Not really, static routes can also be used still. BGP is only *really* needed if all of the following applies to what you are doing:

- Do you have to enforce route policies on exterior connections?
- Do you connect to multiple other domains?
- Are your target domains under independent administration?


### Where Does BGP Live In The OSI Model

BGP unlike most routing protocols, really **isn't** a layer 3 or 2 protocol.

>[!NOTE]
>BGP is build on top of TCP, and hence uses a layer 4 service at the very least. Considering that the database that BGP uses for storing its routing information is mostly maintained within the operating system, BGP is for the most part an **Application Layer** protocol.


## Basics

One important difference between EGP and BGP, is that BGP is built on top of TCP and uses a designated port (TCP 179).

The unit of information when routing with BGP is the `AS_PATH` attribute, which lists a series of ASNs that a packet must traverse to reach a destination. BGP now does basically two things:

- If the `AS_PATH` attribute contains the node's own ASN, then somewhere, a loop awaits. So BGP makes the wiser decision and prevents that path from being advertised into the network.
- To compare paths, the path with the least length (i.e. the least number of ASs to traverse) is chosen. Tie-breaker rules must be implemented to choose a path when needed which we'll discuss later.

>[!BUG] A Common Mistake
>We stress that only when an AS actually receives *it's own* number, we deduce that a loop has happened. Some people wrongly assume that repeated AS numbers in the path list are indication that a loop has occurred.
>It is not hard to see that if the above rule is used for all `AS_PATH` creation, such case never actually happens, so this rule is strong enough for us. The reason that we do not use the other rule, is that we can actually find some uses for paths that have repeated AS numbers, *without* causing a loop. We'll see this when we discuss `AS_PATH` in more detail.

As you can see, BGP only sees ASs in the network, and chooses to remain oblivious about what actually happens inside each individual AS. 

This allows BGP to: 
- Have a much higher level of abstraction about the network and makes it VERY scalable.
- Be *incompatible* with typical routing database storages used with IGPs and such. 

In light of this, BGP has its own, personal routing database, maintained in complete isolation from IGPs. Since there is now effectively two planes of routing decisions for each router, it is fitting to give them a name. BGP uses the name **Routing Information Base** or **RIB** for its own database.

What is in the RIB?, well, see for yourself:

```
BGP table version is 2397575782, local router ID is 128.223.51.103
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter,
              x best-external, a additional-path, c RIB-compressed,
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
Nr>  0.0.0.0          162.251.163.2                          0 53767 14315 174 i
V*   1.0.0.0/24       206.24.210.80                          0 3561 209 3356 13335 i
V*                    94.142.247.3             0             0 8283 13335 i
V*                    193.0.0.56                             0 3333 13335 i
V*                    194.85.40.15             0             0 3267 13335 i
V*                    162.251.163.2                          0 53767 174 174 13335 i
V*                    162.250.137.254                        0 4901 6079 13335 i
V*                    212.66.96.126                          0 20912 13335 i
V*                    140.192.8.16                           0 20130 6939 13335 i
V*                    137.39.3.55                            0 701 13335 i
V*                    4.68.4.46                0             0 3356 13335 i
V*                    37.139.139.17            0             0 57866 13335 i
V*                    132.198.255.253                        0 1351 13335 i
V*>                   12.0.1.63                              0 7018 13335 i
V*                    209.124.176.223                        0 101 13335 i
V*                    91.218.184.60                          0 49788 13335 i
V*                    154.11.12.212            0             0 852 13335 i
<and many, many, many rows more ...>
```

This data was taken from the open source router at the [University of Oregon Route View Project](http://routeviews.org/),you can access a Cisco IOS CLI there and use the command:

```shell
route-view> show ip bgp
```

>[!NOTE]- What Is The RIB Really Showing?
>Each row is a route, which is flagged with a status either as `Valid` or `Invalid` or `Not Found` with a single V or I or N. 
>
>The network columns shows us the destination. As you can see, it only uses groups of addresses with a mask. The next column shows us where to go in order to get closer to the destination.
>
>For now, we gingerly avoid the 3 weird columns and focus our attention on the last column, the "Path" column, which is the same `AS_PATH` attribute that we said, and it contains a list of numbers (particularly, ASNs) that go all the way into a network and terminate with a single `i`, meaning that "Once you reach the `i`, switch from the RIB to the internal IGP database and let the IGP take the packet from there, you are done".
>
>You can see that IOS is simply not showing the destination for many of the entries, this is simply just IOSs simplification of the table, so you won't get a headache. It groups routes with the same destination into the same block and only shows the beginning of the block, so ignoring the `0.0.0.0` destination, which is the "I have no idea where to send this packet, please help!" destination, **ALL** the 16 paths shown above, aim for the exact same destination, i.e. the block address of `1.0.0.0/24`. 
>
>At the moment (August 27, 2022), the router is using the 13th path (from `12.0.1.63`), it is marked with a `>` next to the `Valid` flag in the first column.


### Some Unique Things About BGP

There are some curious differences between BGP and other routing protocols.

- As we said, it is a pure *Application Layer* protocol built on top of TCP. It **only** uses unicast messages and forms point-to-point connections for each of its peers.
- It is neither a Distance-Vector protocol per se, nor really acts like a Linkstate protocol. It is sometimes called a **Path Vector** protocol by some people to make it a separate creature from the two. 
  The way it uses the `AS_PATH` attribute really looks like a distance vector protocol, but BGP traverses *groups* of routers in single hops rather than just one, it cannot remain oblivious about what that groups actually looks like, so it is also in some way a Linkstate protocol.
- Sessions with peers with different AS numbers are called *Exterior* BGP sessions and sessions with routers sharing the same system are *Interior* BGP sessions.


## Well-Known and Mandatory Path Attributes

We discuss *well-known mandatory* path attributes of BGP.

>[!NOTE]
>Well-known attributes are attributes that are required to be implemented by all versions of BGP (there is more than just one BGP version).
>Mandatory attributes must be sent with any Update message that BGP sends.

### `AS_PATH`
We already discussed this one, it is *usually* an ordered list of AS numbers sent by a peer to another peer.

- If a BGP peer detects it's own AS number in the AS_PATH attribute, it drops that update, since it means that somewhere a loop has occurred.
- To send an update to a destination, a BGP peer can add it's own AS number to this path attribute, but **only** if the peer that will receive the update is an external peer, otherwise the destination peer will erroneously think that a loop has occurred.

Here is an example of how AS numbers can prepended while an Update message traverses the network:

![[Pasted image 20220914191451.png | 500]]

Here, the AS number 100 is trying to advertise the address `207.123.0.0/16` into the network, to do this, an update message is sent to each of its peers (it is NOT a broadcast message). When that message leaves the AS, the ASN 100 will be prepended to `AS_PATH` and it will keep on going until it notifies all.

>[!NOTE]- `AS_SEQ` and `AS_SET`
>The `AS_PATH` attribute can actually come in two forms:
>- Either as a sequence (i.e. ordered list of AS numbers) which we discussed above, which is more accurately called an AS sequence path.
>- Or as a set of AS numbers (i.e. unordered).
>These two are distinguished with specific codes in BGP messages, but as to what uses the latter has

#### AS Path Prepending

One of the weird tricks that routers employ when first sending path updates, is *path prepending*. Before we discuss what it is, let's say *why* you would do it.

While BGP has a pretty strong saying about what paths are chosen in the internet, it is not, and SHOULD not, be stronger than what network providers actually want (the same way that a static route completely rules out any decision made by an [[IGP]]).

To give BGP the same flexibility, we need to allow BGP to influence path attributes based on *what* attributes make it into update messages, yet as we said, separate BGP networks are under independent administrations, so we cannot really control path attributes as much as we could with IGPs. 

Path Prepending is one such strategy used here.

For reference, look at the figure below:

![[Pasted image 20221011220234.png|500]]

Imagine that the user wants the path `100 -> 200 -> 400 -> 500` to be chosen, but since the path `100 -> 300 -> 500` is shorter in terms of AS hops, BGP will use that path instead. We can't go on and not advertise this path, since in case of a failure on the former route, any path to AS 100 will be lost as far as BGP is concerned, while one actually yet exists.

We can only make that path "look bad", and to do this, AS 100 will deliberately pad the `AS_PATH`  on that route with it's own AS number until it is at least as long as the path chosen by the user. Doing so guarantees that the path chosen by the user is preferred  by all ASs down the line, since at least one hop needs to bet between the source and destination.

This is why we don't use repeated AS numbers in the list as a measure for finding loops, if we did, we couldn't use this method and AS 300 would drop the path immediately, once again we would lose a potential path to AS 100 for no reason.

#### Prefix Aggregation

TODO

### `NEXT_HOP`
This attribute signals the next hop that the receiver must take to get to the destination NLRI in the message. This has 3 scenarios (Call the advertising router $R_a$ and the receiving router $R_r$):

1. If $R_a$ and $R_r$ are external peers, then `NEXT_HOP` will be the address of the $R_a$'s  interface.

![[Pasted image 20221011221115.png|500]]

2. If the routers are internal peers and the NLRI points to a destination **IN** the AS, then the `NEXT_HOP` is any address that belongs to $R_a$.

![[Pasted image 20221011221149.png|500]]

3. If the routers are internal peers and the NLRI points to a destination **OUT** of the AS, then the `NEXT_HOP` is the address of the external peer that advertised this route to $R_a$ (i.e. the IP address of the router which $R_a$ learned this path from)

![[Pasted image 20221011221212.png|500]]

>[!IMPORTANT] A Potential Problem In The 3rd Case
>Referring to the figure above, the `NEXT_HOP` of `192.168.5.1` advertised to the bottom router might not actually be known to it, since it resides out of AS 509, and AS 2103 which holds it, may not have advertised it to AS 509 yet.
>Here, BGP introduces a wildcard rule called *next-hop-self*, where the advertising router uses it's own IP address as `NEXT_HOP` and when it receives packets for `207.135.64.0/19` it manually sends it to AS 2103.

### `ORIGIN`

This attribute is used as a secondary measure to tie-break paths, it indicates where the path actually comes from in terms of what protocol advertised it in the first place. There are 3 values for it. Based on preference from highest to lowest: 

- `IGP`: Means that the NLRI carried in the route was learned from the internal routing protocol in the originating AS. 
- `EGP`: The NLRI was learned via [[EGP]]. Since EGP is obsoleted, you'll never see it probably.
- `INCOMPLETE`: Anything other than `IGP`.

This attribute might seem unnecessary, and it kind of is. It was used to ease the migration from EGP to BGP, hence it is probably going to be obsoleted soon, but at the moment you'll still see it.


## BGP Decision Process

Now we come to the heart of BGP, how it actually comes to prefer routes to the same destination.

Before that, we should discuss a few things about RIBs.

### Smaller Parts of BGP RIB

BGP RIBs are made of 3 individual databases that feed information to each other.

- `Adj-RIBs-In`: Contains feasible routes learned from peers (i.e. routes that are yet to be checked and processed. While they are feasible, they are yet to be known as optimal).
- `Loc-RIB`: Contains routes that were chosen at the end of the BGP decision process on the routes in `Adj-RIBs-In`. These routes are used to route packets within the AS as well as outside of it.
- `Adj-RIBs-Out`: Contains what routes we shall advertise to the outside domain. Here, *BGP Route Policies* also come into play, that determine which routes get to stay here and which one will be dropped.

The decision process (which we'll discuss shortly) operates on 3 phases using these databases. Note that these steps are performed in full sequence (i.e. the next phase cannot be initiated without the previous phase being completed).

1. Using the decision process, BGP outputs a nonnegative integer for each update message that the speaker receives (i.e. anything in `Adj-RIBs-In`). 
2. First, any route that contains a loop is removed by checking whether or not `AS_PATH` contains the local AS number. Once done, routes going to the same destination are chosen based on the preference metric chosen in phase 1. The results are put in `Loc-RIB`.
3. Outgoing routing policies are performed and results are put in `Adj-RIBs-Out`. If route aggregation is allowed, it is performed during this phase.

Phase 2 has a few caveats. First of all, if `NEXT_HOP`  of some route is found to be unreachable, it is not considered during this phase (though it is not necessarily dropped, it stays in `Adj-RIBs-In`). This phase also always chooses the most *specific* route (we'll show what this means later).

### The Decision Process

Implementations differ slightly, but they are not unimportant, so make sure you know what you are using. We move on to step $n+1$, if and only if step $n$ has tied. We have not yet discussed all the details (namely, BGP Confederations), but a general idea is best given now instead of later.

Generally:
1. Prefer the route that were injected directly by the user. If none are found, prefer routes that the router itself injected into BGP.
2. Prefer routes with shorter `AS_PATH`.
3. Prefer based on `ORIGIN` preference.
4. Prefer based on lower `MED` value (Multi-Exit Discriminator). This value is essentially a weight, used to disambiguate same paths that enter their ASs on different peering points. When each AS uses a single IGP, the metric value generated by the IGP is used for `MED`, otherwise, it needs to be manually configured, or just ignored.
5. Prefer EBGP routes, over EBGP Confederation routes. Prefer EBGP Confederation routes over IBGP routes.
6. Prefer the route with the shortest path to `NEXT_HOP` using data from the IGP protocol.
7. Prefer the path that was received first. This can help lower the amount of actual changes to network routes, unless newer router can won at any point above.
8. Prefer the path with the lowest BGP router ID.
9. Prefer route advertised by the neighbor with the lowest IP address.

There are steps at the beginning and some steps at the end that we omitted for simplicity.


## BGP Message Formats


