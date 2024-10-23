
**Title:** Mitigating the Performance Impact of Network Failures in Public Clouds
**Link:** [Arxiv](https://arxiv.org/pdf/2305.13792.pdf)
**Conference:** NSDI 2024

# First Pass

The paper is about the design of a system that allows data center administrators to know what to do when a failure happens in the network and how to best mitigate it.
To provide context:
- Most DCs have a huge number of switches and active components, with servers producing a huge amount of flows into the network. At such a scale, failures become much more common and the administrators need to make decisions about how to mitigate them.
- The state of the art for mitigating these failures are simple heuristics, which improve performance only *locally*, they do not consider the impact of their changes on a wider level, for example:
	- When loss is detected on a link, draining it and replacing it immediately is not necessarily the best action. If the loss is tolerable, it is best to keep this link for some time.
	- On the other hand, if a link is known to be in a shoddy state, routing more traffic through it to alleviate congestion might lead to failure and actually decrease the performance!
The system under question, SWARM, tries to reason about this, and provide a more optimized action in the face of failure. 
The results look very promising; For example, comparison with [[NetPilot]] show a huge benefit (up to 700 times) in flow completion time.

