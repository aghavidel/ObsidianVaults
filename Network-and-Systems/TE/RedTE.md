**Paper:** [RedTE: Mitigating Subsecond Traffic Bursts with Real-time and Distributed Traffic Engineering]([RedTE: Mitigating Subsecond Traffic Bursts with Real-time and Distributed Traffic Engineering | Proceedings of the ACM SIGCOMM 2024 Conference](https://dl.acm.org/doi/10.1145/3651890.3672231))
**Where:** ACM SIGCOMM 2024

# Intro.

This paper proposes a distributed Traffic Engineering (TE) solution.
The motivation here comes with noticing how TE is done over Wide Area Networks (WAN) currently. As we have seen in the context of [[CS656]], WANs are always:
- *Overprovisioned*, to tolerate failures, and as noted in this paper, to tolerate **Bursts of Traffic**
- *Minimally Optimized*, they are expensive as-is, span large areas (duh ...) and it is hard to control them. 
  Another reason is also the fact that you cannot introduce new things into them that easily, because a lot of the devices within them take a long time to replace, so they tend to lag behind a lot of the new technologies that we may have.

As noted in the paper, provisioning also follows heuristics that have a high chance of being too pessimistic. For example, [here]([Best Practices in Core Network Capacity Planning White Paper - Cisco](https://www.cisco.com/c/en/us/products/collateral/routers/wan-automation-engine/white_paper_c11-728551.html)) it is noted that many WANs are provisioned by doubling link bandwidths the moment that the average utilization goes above 50%.

>[!FAQ] Why The Above Works?
>Simply put, most WANs have service level agreements that involve both aggregate bandwidth and delay, but it turns out that with most traffic samples, you can just boil it down to bandwidth.
>This is because (at least as far as Cisco seems to have seen), most traffic in the WAN obeys a Poisson distribution (or at least is Markovian), which means that queuing delays drop by increasing bandwidth.
>In contrast, other traffic models (say, self-similar traffic that we discussed in [[Self-Similarity of Ethernet Traffic]]) will not follow this behavior (i.e. just increasing bandwidth won't work, you need to prevent heavy aggregation of traffic).

This paper is an attempt to prevent massive bursts using careful TE design. 
The main difficulty here though, is that TE takes time. Most solutions, like [[DOTE]], [[TEAL]] and [[NCFlow]] operate on time scales of a few seconds and larger, whereas bursts happen within milliseconds for a single flow and go up to a few hundreds of milliseconds when flows are aggregated.

TE is slow for many reasons:
- *Computation*: Most TE problems turn in to some form of a Multi-Commodity Flow (MCF) problem. Exact solution to these take just too much time, so approximate solutions are used.
  This isn't the biggest concern for us, since computation can be scaled efficiently within data centers, and efficient heuristics can also be used.
- *Data Collection*: TE solutions need feedback. In particular, they need measurements from remote switches. This is even worse when considering a WAN, where collecting data from routers over a wide area can take a huge amount of time. To this end, we need to be careful, since if this loop takes a few seconds, we would be effectively blind to when or how bursts happen.

The question is, how can we lower the delay on each loop?
One answer is to give routers more intelligence. Instead of aggregating the entire TE optimization into a single, centralized unit, we need to incorporate some of it into individual routers, and in doing so, we would be able to make local decisions specifically to mitigate bursts.
This of course has a potential drawback, since routers do not know much beyond their neighbors, and if they make drastic decisions, we'll be in trouble!

In essence, we face a *distributed* TE problem (dTE), which is not unique (MPLS-TE is probably the most well known type of dTE), and we are asked how to adapt it in such a way that it can detect traffic changes in a scale of a few tens of milliseconds. 

# Background

As we have been told many times, internet traffic is very bursty. That was the case with ARPANET, it is still the case after all these years. 
Bursty traffic tends to be a bigger problem now more than ever, since we are rapidly starting to deploy infrastructure for applications that are heavily *delay-sensitive*, things like online trading, AR/VR or autonomous vehicles. These applications need guarantees on their delays, and we truly don't have a good way of doing that currently.

The most widely used method, is heavy over-provisioning, which is of course far from optimal, but TE is a solution that was never associated with this problem.
In practice, TE does not tend to consider delay as part of its objective function, people generally assume that as long as bandwidth constraints are met, good congestion control will minimize delays by reducing queues on each router.

The above does not apply when you have bursts, unless you keep significant free bandwidth on each link, so we need to think about this a lot more.

## Current TE Systems

Almost all TE systems that we have designed until now, boil down the problem into a MCF problem, in which the input is a demand matrix $D$ where $d_{s, \epsilon}$ is the demand from source node $s$ to the destination node $\epsilon$. The problem is often formulated based on paths instead of edges, as such, for each pair of source and destination nodes, the paths are chosen beforehand.
The optimization will then output the fraction of traffic to be sent on each path $p$, $w_p$.

The problem can be formulated to achieve different objectives. For example, one that is frequently used is *Maximum Link Utilization* (MLU).
When we consider this, most available solutions cannot keep utilization below 50 percent when bursts happen (the solutions being [[TEAL]] and [[DOTE]]).

![[Pasted image 20241019030742.png|600]]

In the above, the burst ratio is defined as the ratio between average traffic of two adjacent slots of 50 millisecond each. As you can see above, traffics are extremely bursty, as such we cannot disregard this.