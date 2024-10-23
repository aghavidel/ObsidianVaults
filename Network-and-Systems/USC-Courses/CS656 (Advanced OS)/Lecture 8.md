**Paper:** [Cores That Don't Count](https://dl.acm.org/doi/pdf/10.1145/3458336.3465297)

# Intro.

Until now, we blamed failures on the developers, on the power, and on the software, so what about the hardware?

For example, **what to do if the CPU itself is not reliable?**
From a distributed system perspective, until now, all the failrus that we have considered have been **Fail-Stop**, the system just stops functioning and leaves the topology.

We should start considering **Byzantine Failures** now, where the system isn't going to stop, it is going to produce output that is *faulty*.
CPU failures here is some form of Byzantine failure, where the CPU seems to work, but is not executing commands *correctly*. 

This is nefarious, since at scale, no one is monitoring the results of these executions, so it can take a very long time until we realize that the CPU is generating garbage. We call these trouble maker cores **Mercurial Cores**.

So, there are 3 things we can consider at this point:
1. Is there a **proactive** way of dealing with them? Making sure we mitigate them before they actually start messing something up
2. Is there a concrete way of detecting them?
3. Can we hope to just live with them?

## Proactive Detection

We don't yet know what exactly causes a CPU to become mercurial. If we can develop a model that predicts when a CPU becomes mercurial, then we can develop test suits that can detect that, and run them at the time when we thing a problem could start to manifest.

We can also set a lifetime for each device, and then throw it away when that time expires!

## Detection

We can run test suits on each core to tell us when things are wrong!

## Living With Them

Triplicate each computation result, and take majority.
