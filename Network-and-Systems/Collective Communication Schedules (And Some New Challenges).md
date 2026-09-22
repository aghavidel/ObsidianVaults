# Intro.

Collective Communication (CC), broadly speaking is any scheme that invokes a communication function across multiple nodes at once.

In our current landscape, there is really only 4 collectives that you hear about often:

- `All-Gather`

![[Pasted image 20260921220833.png|500]]

- `All-Reduce`

![[Pasted image 20260921221413.png|500]]

- `All-to-All`

![[Pasted image 20260921221639.png|500]]

- `Reduce-Scatter`

![[Pasted image 20260921222651.png|500]]

People use CCs in many different ways, but for the context of this talk, CCs are limited to the case of LLM training or inference (with a bit more detail, we'll use the way that Megatron uses them).

## When Are They Used?

- The classic `All-Reduce` usage is for Data Parallelism (DP) for syncing the gradient of each data rank by taking an average.

![[Pasted image 20260921223807.png]]

- `All-Reduce` is probably the most researched CC algorithm in terms of schedule as:
	- It is heavily susceptible to stragglers
	- It is _ubiquitous_, almost every model uses it (even during inference)
- Significant work has been done to see how folks can mask the overhead of `All-Reduce`, and hence it has produced probably the most diverse set of algorithms:
	- **Ring All-Reduce** for _large_ messages
	- **Double binary tree** for _small_ messages
	- **Hardware offload** (e.g. gather, reduce in-network, then scatter)

- There is an endless array of heuristics in use that try to determine which algorithm is best suited to a setting.

# Are They Good Enough?

Nope! They suffer on two fronts:

- **Algorithm Selection Heuristics are Too Simple:** Many of these algorithms analyze message size and topology *without assuming something is already using the network* (be that another collective, or even the same collective).
- **They Are Topology-Agnostic By Design:** NCCL is only informed of the topology during start up. For this reason, the algorithm templates do not assume anything about topologies or paths. For this reason, they forego using symmetries or which particular points of a topology make good aggregation points.

## Example: TACCL

Here, an `All-Gather` operation is being tested against NCCL.

![[Pasted image 20260921233747.png]]

>[!FAQ] Why Is NCCL _THIS_ Bad Here?
>The topology in this setting looks like this:
>
>![[Pasted image 20260921235222.png]]
>
>NCCL uses a 2D ring in this setting:
>- 1 ring per node for intra-node data communication
>- 1 ring between nodes for inter-node communication
>
>NCCL by default moves data chunk by chunk in lock-step. Thus at each step, 1 chunk is moved within an intra-node ring, then it is sent over to the other nodes over the inter-node ring. This is very inefficient, as it means that basically one of the 4 pair of NICs is being utilized at all times.
>You can now probably see why TACCL beats NCCL by a factor of 4 for large messages.

# Custom Schedules?

Many companies (your Google's and your Microsoft's) use custom NCCL backends engineered for their networks. Deploying custom scheduling algorithms directly on top of NCCL is quite a pain.

Prior research at this point has successfully been able to soften this barrier. In particular, MS-CCL, which used to be Microsoft's internal CC library has a DSL that lets you implement virtually any collective schedule (there are caveats to this ... but they don't matter for us).

So focus has shifted mostly on just determining what that schedule should be to begin with?

## Towards Optimal Schedules

TE-CCL from SIGCOMM 2024 was one of the papers that showed that it is very possible (although incredibly slow) to generate optimal schedules using prior knowledge of the network and the collectives.

- It established that tractable (but slow!) modeling of the problem can be done optimally by using similar methods that people use for Traffic Engineering.
- But it is still a much harder problem!

### Why Is It Harder Than TE?

It breaks many constraints that TE cares about. One big one is flow conservation:

![[Pasted image 20260922002231.png]]

In the above, `h` can make copies of the chunk it receives and send them concurrently on the output links for the optimal 2 second completion time.

This is also a setting where we need to explicitly account for capacity constraints *over time*, meaning that when a collective starts, it does not immediately load every hop on the paths that it uses:

![[Pasted image 20260922002854.png|500]]

In the above, if the green and blue chunks are small enough, we can optimally schedule them such that they never actually have to share the link `h3 - d`, but that statement wouldn't be true if we ignore link delays.

>[!IMPORTANT] This Is Actually The Most Important Thing
>Not correctly accounting for congestion due to link delays is the exact reason that NCCL collectives can be treacherous for small message sizes (and in many of the "Optimal Schedule" papers, the largest gains happen when collectives use small data sizes)

>[!FAIL] The Trouble!
>While explicitly taking care of latency can give nicer solutions, it comes at a heavy cost, that being that we have to consider **multiple snapshots of the same topology for each collective**.
>For example, we may have to create replicas of the topology for each micro-second and solve over many of them at once. Thus, these optimal scheduling problems generate many many variables.

The result:
- 2 hours of runtime for just a 16 node topology
- Gives better results, but not by much (look at the 64 KB result for comparison)

![[Pasted image 20260922003931.png]]

## OptCCL

This paper came out this SIGCOMM, and for most intents and purposes, gives the most proper solution to optimal scheduling of collectives with known networks.

![[Pasted image 20260922004333.png|500]]

>[!FAQ] How Is It This Fast?
>OptCCL isn't really "optimal", there is a tiny gap between its final solution and the "actual" optimal, but here is not good reason for us to be worried about it. Accepting this error however allows is it use some well known decomposition methods that allow it to speed up its final solve.

# Are We Done?

>[!SUCCESS] Is This Problem Solved?
>Optimal collective scheduling is pretty much a solved problem for:
>- **Collective With Known Size**
>- **Fixed Networks**
>- **Assuming any schedule is permitted**

>[!FAIL] It Is NOT Solved For
>- **Collectives With Sizes Not Known Prior**
>	- **Example:** `All-to-All` collectives used for MoE
>- **Changing Networks**
>	- **Example:** What to do if a path fails or a GPU goes out?
>- **Changing schedules has an overhead**
>	- Deploying a weird and complicated schedule over hundreds or thousands of GPUs is very non-trivial, even assuming that each node has basically cached all prior schedules.

# Special Case: Optical Networks

One case where CC is still very unclear (and needs to be cleared up sooner rather than later!) is for topologies that use *Optical Circuit Switches* (OCS) instead of packet switching devices.

>[!REMINDER] What Was an OCS?
>![[Pasted image 20260922010424.png|500]]

## Why Do People Like OCS Now?

The main draw of optical circuit switches is that:
- They are **very power efficient**
- Very durable
- They trivially scale up to virtually any input traffic that you give them (they don't switch packets, they just pass-in light regardless of line rate)

>[!FAQ] How Much Power?
>- Nvidia's cutting edge IB switch uses **more than 2000 Watts**.
>- A typical OCS **barely uses more than 100 Watts**.

More importantly, OCS devices had two major draw-backs that are being dealt with efficiently now:

- Fibers are flimsy! They break and disconnect.
- Long fibers require amplifiers that use up extra power.

People now have many ways to handle both of these shortcomings (the MOSAIC paper from SIGCOMM 2025 is a good example about tackling these issues).

## GPU Clusters With OCS?

The promise of power efficiency and not having to really care about NIC speeds is nice, but there are many aspects of OCS devices that are a cause for concern:

- **OCS Circuit Reconfiguration Is Slow** (it can go above 100 ms)
- **Number Of Paths Are Limited**
- **Reconfiguration Disrupts On-going Flows**

None of the above are nice things for CC (which typically end within milli-seconds)

>[!FAQ] Optimal (Even Acceptable) Schedules With OCS?
>Little work has been done on this front.
>The closest work is HARVEST from SIGCOMM 2026 (FYI that last two best paper awards went to papers concerning OCS devices), but it **assumes that collective schedule is already known**?
>
>Optimal circuit and collective schedule to my knowledge is unsolved for now.
