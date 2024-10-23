# Intro.

In this note, we first describe traffic engineering and it's general place for network optimization, then shift our focus to arguably the most popular way of implementing it, which is in combination with [[MPLS]].

## What Traffic Engineering Means?

There is basically two main faces to the whole "engineering" aspect of networks. 

- **Network Engineering:** Which means we change the network (in any different abstraction of it) to fit our needs. Perhaps we need to guarantee some QOS in the network, or put some constraints on how traffic is routed to certain destinations.
  One of the main things about network engineering is that it is generally a very lengthy process, as it usually entails installing several new and sometimes complicated hardware in the network. So it is pretty obvious that it is NOT going to work for everything.

- **Traffic Engineering:** This is what compliments network engineering in *most* scenarios. We ***change the traffic*** to ***adapt to network inefficiencies***.
  You might be asking "how would you change traffic, a completely external parameter?", and you would be correct, we don't really change the traffic, and as with any other optimization solution, there are cases in which it completely breaks down. The better (and more mouthful) wording of this might *Traffic Distribution Engineering*.
  To make this more apparent, let's start with an example of how TE would be used.

>[!EXAMPLE] Basic Example Of TE
>Imagine you want to route traffic through source $S$ to destination $D$. You have two paths on to choose, path $P_1$ and $P_2$. You are using a simple [[IGP]] like [[RIP]], but you *know* somehow that path $P_1$ provides much higher bandwidth compared to $P_2$, how would you force traffic through this path instead of $P_2$?
>One solution of course would be to use more complicated IGPs, but that is costly and not very feasible. The more sensible choice would be to influence link weights in such a way that $P_1$ is always preferable to $P_2$, you can use administrative weights on links to overrule any other metric and so do what you want!
>There can be even more "smart" ways of doing it, like splitting traffic between links such that we achieve some state of *fairness*, like splitting with equal utilization, but it is not really obvious how one would actually do that.
>We are not changing protocols, networking equipment or network topology, we are merely influencing things in such a way that it behaves as we want it to. This is what TE does.
>

So you might ask how MPLS of all things would make such tactics easier?

It might be insightful to see what people did *before* MPLS to achieve TE for that … . For the most part, the main to ways that TE was done was either with **IP** or **[[ATM]]**.

### TE With IP

IP-TE is simple and still very much used, but it does not have control on the more nuanced and small scale distributions of traffic (which if present in enough quantities, can change the outcome of the engineering process).

In simple terms, IP-TE is what we did in the previous example, manipulating link weights based on certain factors (like destination IP) to control traffic flows. We'll soon see what problems this approach cannot solve, but for the most part, given a fast enough computing source (which is not very trivial to design of course), it is possible to achieve most TE goals with just this simple approach, but there are limits, which are exasperated when different protocols need to co-exist in the same network and parts of the network are under separate administrations.

### TE With ATM

ATM is probably one of the more lavish approaches of TE. It basically means to literally force traffic only on certain desired paths (possibly with total disregard to what IGPs are outputting) and reconfiguring them based on how under/overutilized certain paths are. 
ATM is used, since it allows ISPs to place permanent virtual circuits (PVCs) for each desired path with ease. 
Once again how we would monitor path resource utilizations is also a non-trivial answer, but as we shall see, [[RSVP]] gives some solutions for that. But that aside, one of the main downsides of ATM-TE which has caused it's steady decline of use in large networks is that is very bad at adapting to new network situations, particularly topology change and failures.

It is not hard to see that such an approach to TE can result in full-mesh networks of PVCs (since PVCs are chosen independent of each other) and this means that in order to flood link-state data into a network of $N$ nodes, we need to:
- Flood $\mathcal{O}(N^2)$ messages in case of link failure, since a message will be delivered across every remaining link at least once.
- Flood $\mathcal{O}(N^3)$ messages in case of node failure, since each will have to inform all other nodes about it, meaning that we'll have $\mathcal{O}(N).\mathcal{O}(N^2)$, so $\mathcal{O}(N^3)$ messages flooded into the network.

### An Example (Fish Problem)

Consider the following network (also called a *fish*):

![[Pasted image 20221101004234.png|400]]

We want to send traffic from $R_1$ and $R_7$ to $R_6$ (i.e. from the tail of the fish to its head). Using normal strategies of routing, the path would be from $R_2 \rightarrow R_5 \rightarrow R_6$ regardless of who sends the traffic, since it just has lower weight compared to the alternative. 

The problem is that there is no reason to assume this path can handle two active streams at once, and we might end up with never ending queues in $R_2$. We need some mechanism of load balancing.

Of course, load balancing is a pretty well developed topic, and one could use ECMP (Equal Cost Multipath) routing. This can be done by setting the weights of both $R_2 \rightarrow R_5 \rightarrow R_6$ and $R_2 \rightarrow R_3 \rightarrow R_4 \rightarrow R_6$ to be the same, and letting $R_2$ balance the load of $R_1$ and $R_7$ across both paths, problem solved!

Only there is a little problem, what if instead of just one path, there were hundreds, and what if instead of a single edge router ($R_6$) there was also a hundred of them? The problem turns into assigning weights to a graph such that a given set of paths have the same weight. This is really not an easy problem to solve, in fact, unless we allow zero weights, it may not have a solution at all (though that requires some weird paths to be chosen, but still …).

This is for the most part, infeasible to solve in such a way with our current IP-TE approach.

On the other hand, ATM-TE just trivializes this problem, just install two PVCs with the same cost to each destination and use literarily any load balancing scheme to achieve acceptable performance (we won't talk about "optimal" performance yet, since that requires a lengthy definition of what optimal performance actually is).

But as we said, ATM-TE has its own problems, we can probably do better right?

## MPLS-TE

