# The Tail at Scale (Again!!!)

Not to get in too much detail.
Google, which is the place where this paper comes from, heavily relies on *fan-out* communication patterns to heavily parallelize communication among a set of nodes. 
Of course, every single request, has variability in terms of latency, so every RPC has a distribution for its delay $\mathcal{P}_\tau$.
Now, as you learned from elementary probability, the CDF of $\max{X_i}_{i=1}^{i=n}$  if $X_i$ are independent would be the product $\prod\limits_{i=1}^n \mathcal{F}_\tau$  , which means that now, the tail is found by looking at the value of $p^\frac{1}{n}$ on $\mathcal{F}_\tau$, which would be just terrible if the tail latency of $\mathcal{F}_\tau$ deviates heavily from the mean.

![[Pasted image 20241022122346.png|600]]

So, the goal of this paper is to show:
- How exactly tail latency wrecks system performance
- Why tail latency would be high
- How to reduce tail latencies
## What Causes Tail Latency

Many sources:
- **Resource Sharing:** (The main one) servers trying to reach the same server to grab a resource. Schedulers (especially thread schedulers!) can always behave unpredictably and cause late responses.
- **Queue Delays:** Which we have seen a lot. This can get worse if we have tasks that vary widely in terms of size, which can cause head-of-line blocking.
- **Hardware Noise:** Hardware can be unpredictable at times ... CPUs can also get tuned down because of power fluctuation or getting too hot

### Hedging Requests

If tail latency is just high in general, what we can do is to issue requests to multiple replicas and get the one that arrives fastest (of course, you might need at-most-once semantics).
Of course, the naïve way of doing this would be extremely expensive.

So a better way of doing hedged requests, is by *waiting* a little bit:
- Issue a series of requests, but wait until sending the hedged one
- How long to wait? Depends, but an example would to be wait for the p-95 percent of the latency. If you did not get the response, send the hedges.

## Handling Queuing Delays

Queue  delays are a big headache, since they can be extremely variable. 
To minimize queue length, *how you choose replicas for sending requests* is quite important. One way to handle this is classic "power of 2" scheme. Choose 2 servers at random, inspect their queue lengths, and then send your request to the least busy one.

Another way is also to *tie requests*.
- Send multiple requests to servers
- The first server that starts the executions sends a quick `CANCEL` RPC to other servers

This only works if the servers are in the same datacenter so that we can send cancel RPCs quickly. So something like a Paxos replication system cannot be used with this.

### Canary Requests

Sometimes, if a system is executing untested code, then it is much safer to issue requests to only a small set of servers and to test if they explode or not. We send other requests if and only if the servers did not explode.


"LOOKUP DIFFERENCE BETWEEN ATOMICITY/SERIALIZABILITY/STRICT SERIALIZABILITY"
