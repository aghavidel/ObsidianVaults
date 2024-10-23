**Paper:** [Cluster-Based Scalable Network Services](https://dl.acm.org/doi/pdf/10.1145/268998.266662)
# Cluster-Based Services

Last time, we discussed performance improvements, but now, we are going to look into the OG paper that actually really proposed cluster-based services. We are going to start learning what are the main things that need to be considered for these cluster-based systems.

We want from these systems to:
- **Highly Availability**, be up even if services are down
- **Scalability**, be able to dynamically increase the number of servers instead of having to buy larger servers. Usually, this is defined as having the property that when a system grows in scale of size $N$, then its performance penalty increases at most linearly with $N$. 
- **Cost Effectiveness**, be fault-tolerant and be able to do incremental upgrades.

Clusters can achieve this, as long as there are constraints on the inputs, in particular, the traffic should be:
- **Embarrassingly Parallel**, in that we receive a lot of requests that have nothing to do with each other
- **Independence of Responses**, in that a response does not rely on shared state between particular nodes in a cluster
- **Approximation Tolerant**, if some subset of servers are down, then use some approximate of the result from available servers to answer a query.

These 3 give us 2 very important properties that make cluster good:
- **It is not important which server handles a request**
- **It is not too big of a problem if query result is not as good, as long as we can answer it in some way**

Another big utility of this system also is dynamic scaling:

![[Pasted image 20240415104914.png|500]]

The above model is referred to as a Scalable Network Service (SNS) model.

In particular, all workers and frontends can be scaled up or down based on the load. Most systems rely on ACID semantics (Atomicity, Consistency, Isolation and Durability), but some clusters also implement the BASE semantic (Basically Available Soft state eventual Consistency), which means that the system is available and **eventually** consistent.
This means that if a failure happens, we can put up with approximate answers, but eventually we will converge to correct answers when the failure is handled.

There is also soft-hard state distinction. 
- **Soft State** is a state that can be reproduced if it is lost
- **Hard State** is a state that if lost, cannot be recovered

Basically, we want ACID semantics with Hard state, and BASE semantics for Soft state. In the hope that most of the system state is soft, we can make the system more scalable compared to a full ACID implementation.

This is where **Statistical Multiplexing** becomes a key thing to consider, however, the paper originally proposed to have extra servers in the pool and power them up when the manager notices that things are getting hot.

## Load Balancing

Load Balancing is a very important thing to consider in a cluster, since that determines whether or not the cluster becomes bottlenecked on itself. 
We cannot rely on a centralized load balancer, since if it goes down we are screwed. So most systems just query the load of workers when getting a request and forward to the least loaded one. This is an approximation but works. 

Another way is to pick 2 random workers and forward to the least loaded one on that set.

Today, this practice is the main thing that makes clouds attractive, they will implement load balancing and fault tolerance, and the application developers need only focus on their own application.

>[!IMPORTANT] Example of Applications That DO NOT Work With Clusters
>1. If the system needs very tight ACID semantics, then just using a data center is much better than a cluster.
>2. Multiplayer games! It is not that easy to completely parallelize them.
>3. Most things that use money! Some central state or consensus core is needed to make sure that things are not screwed.
>4. Any system that cannot tolerate eventual consistency, and requires total consistency or service denial if that cannot happen. Stock market for example is one of these, if the server state is so bad that it cannot answer some queries completely correctly, then it is best not to answer and deny it.

## Evaluation



