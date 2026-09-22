>[!FAQ]- LaTeX macros
>- Predecessor of node $i$: $\newcommand{\pred}[1]{\text{Pred}(i)}$
>- Per iteration value: $\newcommand{\atiter}[2]{#1^{(#2)}}$

This document tries to (in)formally describe the current model training straggler detection algorithm that Mahdi has developed. This is mostly a high level overview target specifically at the author (mostly for the purpose of writing a TLA+ module for it).

# System Setting

- We operate on a system with multiple hosts (nodes) interconnected over an unknown underlay topology. The intra-node underlay topology for each node is assumed to be a mesh (implemented either via PCIe or NVLink). Each node can contain multiple GPU devices which perform computational tasks.
- The logical topology (i.e. the one that governs the communication patterns between GPUs) is assumed to be some sort of ring or tree (or compositions of them.
- We will first begin by describing the mitigation method for a logical topology that is just a single ring with $N$ nodes. GPUs can be numbered from $1$ to $N$.
- Given the current logical topologies, each GPU index $i$ has exactly one parent node which we denote as $\pred{i}$.

## Stragglers (and their definition)

NCCL operations on a collective level are asynchronous, but barriered. Meaning that all ranks participating in the operation must arrive at the barrier for the operation to be declared complete. This is the exact reason why stragglers become such a problem, as one GPU slows down the entire system.

Reasons for why a node becomes a straggler are diverse, the ones that this author thinks apply to our case though are:

- Faults in the GPU itself (e.g. some cores being faulty and unavailable to CUDA)
- Clock throttling because of high temperature
- CPU overload (kernels are executed on the GPU, but *launched* by the CPU, thus a slow CPU bottlenecks overall execution on the GPU)

What we *don't* consider for now in this document are communication related stragglers (faults in the NVSwitch, PFC deadlocks or slow-downs, faulty tuning of GPU-Direct, etc.)

# Modeling Training Iterations

For our intents and purposes, we are only interested in the case where model training is being done over NCCL. This particular case gives us many properties that we can use, most importantly:

>**Since training is naturally done over repeated iterations, the computation and communication patterns within an iteration do not change.**

As an example, assume that we are training over two GPUs with DP. Each training iteration consists of:
- Concurrent forward and backward pass on each GPU
- An `AllReduce` collective
- Concurrent optimization on each GPU

>[!NOTE] Background on NCCL Kernel Launches
>When a NCCL collective is to be executed, individual NCCL processes on each GPU attempt to launch operations aggregated as a single CUDA kernel launch (usually with `ncclGroup` semantics).
>
>The CPU schedules the kernel launch and the GPU (possibly with some delay) executes the kernel. While the kernel is being processed, NCCL channels may attempt to send or receive data concurrently as the computation is being done (NCCL basically interleaves compute and communication over its thread blocks).
>
>NCCL communicates data in *chunks*, this is the smallest unit of data that can be communicated over channels. The size of each chunk depends on the hardware specification and NCCL heuristics. It is not easy to predict it seems, so we should not rely on it that much.

In this model, each training iteration naturally consists of periods of compute and communication:
```
< (Concurrent) Compute > < Collective > < Compute > < Collective> ...
                        |                          |
                        v                          v
                       T_1                        T_2
```
The concurrent compute usually consists of kernels created by ML frameworks (say, `PyTorch`), but the collective kernels (which can have small computation kernels within them, like reductions and such, but can never too much), and collectives are handled by kernels created by NCCL which implement collectives like `AllReduce`, `AllGather`, etc.

## Describing Each Iteration

Take the setting where we have $N$ GPUs indexed with $n$, let us focus on one iteration indexed with $i$. Each iteration has known, fixed amount of collective operations that it must execute like $M$.
At the end of training iteration $i$, each GPU $n$ will report a list of $M$ timestamps $\atiter{T_{nm}}{i}$ for $1 \leq m \leq M$ in ascending order. One can naturally also compile this into a single $N \times M$ matrix per iteration for convenience.

Per the assumption that collective patterns do not change, the entire training procedure can be fully described by just specifying each matrix $\atiter{T}{i}$ for all $1 \leq i \leq I$. If we actually have the matrix at each iteration, the task of finding stragglers is equivalent to finding one or more rows in the matrix that 

Individual collectives spawn kernels on each GPU for executing the operation. For our intents and purposes, we cannot make use of any collective level metric (i.e. when a collective starts and ends), we need per-GPU data.

Assuming NCCL groups all kernel launches into one call on each GPU, we need only focus on what individual kernel launches look like on a single GPU to model the interactions. Since kernels interleave compute and communication, we can just model them as completely asynchronous nodes with input/output queues and arbitrary delays.
```

==<Input Queue>==> (GPU compute) ==<Output Queue>==>

```
We assume we can collect even timestamps for each of these elements, i.e.:

- For each push into the input queue of GPU $i$, we get an event indexed by $m$ at time stamp $I^i_m$.
- For each pop from the output queue of GPU $i$, we similarly get $O_m^i$.
- The GPU reports a single compute time for each kernel $k$, which will be denoted as $C_k^i$,

**NOTE: Talk to Mahdi about what we can actually get from NCCL, this is probably too much ...**  

It is important to note that the dependence of these values isn't known to us at all. But we do assume that since we only deal with compute stragglers, localizing the GPU with the highest kernel compute time at each iteration might prove helpful (i.e. highest $C_k^i$ for some horizon).

We also further assume that at each instance, each GPU node reports for us the number of received chunks aggregated over all channels, which we denote with $L^i(t)$. This is obviously a function of the previous values, but we treat it as a black-box variable here (i.e. it is an oracle variable that we just query when we need and assume it returns something reasonable).

>[!IMPORTANT] Clock Synchronization
>Since all operation happens within a datacenter, it is possible for us to assume that we have very tightly synchronized clocks among the GPUs.
>As such, we assume it feasible to not only query values (especially things like $L^i$) at any particular time locally, we can also query them for a particular time *remotely* (i.e. node $A$ can ask a remote node $B$: "How many chunks had you received at time $t$?").
>
>Having such finely synchronized clocks prevents many race conditions from happening later in our detection algorithm.

# Detecting Stragglers

If all of the GPU level timestamps that we described above and the chunk counts can be collected in one place, the problem is obviously easier to solve, but given Google's specification to make sure that we are not intrusive and we don't want to induce too much overhead, we have converged on a distributed method instead of a centralized one.

This means that each metrics (i.e. $I$, $O$, $C$ and $L(t)$) are attached to GPUs and we can only query them. Each GPU instead has an attached user-space level process (optionally in a separate NUMA node) that is bound to it, able to query metrics directly, but must communicate with other processes on other GPUs either with IPC or message passing of some sort.

All local information at time $t$ can be collected in a single struct which we denote as $\text{LocalMetrics}$. This struct also contains a timestamp $t$, which describes when the data was sampled on the original source.

We refer to each of the user-space processes that have been bound to a GPU as a *Monitor* process (obviously we can index them with the GPUs themselves).

While this is nice from a data collection perspective, it does make us vulnerable to the danger of *false stragglers*. Basically, since each process may now be able to flag stragglers individually, it may be possible that some non-straggler GPUs can be flagged as well during the detection procedure (this happens since the logical topologies have only a single data path between each node by definition, and thus just from local metrics, it is impossible to distinguish stragglers on the upstream from normal nodes).

For this reason, this distributed method will have two subroutines:

- `flagPotentialStraggler() -> int`
	- A non-deterministic procedure that means that a monitor process may claim the existence of a straggler and trigger the next procedure.
	- The procedure returns a GPU node ID (usually, the return value is just assumed to be the predecessor of the GPU that the monitor is bound to on the logical topology).
	- We will not discuss this procedure in this document, but we do assume a few things about it. In particular:
		- The procedure is **sound**, in the sense that no execution exists where `flagPotentialStraggler()` happens and a straggler node is *NOT* present.
		- The procedure is **causal**, in the sense that if monitor process $i$ completes it at time $T$, then at least one of the following holds:
			- Monitor $\pred{i}$ is attached to straggler GPU
			- Monitor $\pred{i}$ executed `flagPotentialStraggler` at some point $T'$ such that $T' < T$.
	- For the purpose of our discussion, the procedure is completely local and thus appears as basically an atomic step. In other words, the body of this function references no variables outside of the local GPU metrics described above and no message passing to other GPUs happens.

- `localizeStraggler()`
	- This procedure is executed by any monitor process that executes `flagPotentialStraggler`. This is the only phase where message passing of some sort is allowed between monitor processes.
	- Monitors may execute RPCs in this phase to communicate with each other. All RPCs are assumed to be completely synchronous and blocking for this phase.

## Description of `localizeStraggler`

We define the following constants for each monitor process:
- `MONITOR_ID: int`
	- The ID of the current monitor process executing the program thread.

Each monitor process independently implements this procedure. The procedure maintains the following definitions:

- `potentialID: (int | None)`
	- This is a state variable maintained for each process.
	- It is the return value of the `flagPotentialStraggler` procedure. It is a GPU node suspected by a monitor process to be a straggler.
	- This variable being not `None` is proof that the monitor process executed `flagPotentialStraggler`.
	  For our intents and purposes, this value will always be $\pred{i}$ if it is not `None`.
	- In Mahdi's phrasing of the algorithm, this state variable being not `None` is equivalent to a monitor process having a candidate straggler node.

- `stragglerFound: bool`
	-  A state variable that signals whether or not a straggler has been found.
	- By default, `localizeStraggler` will keep backtracking until it reaches a state where this flag is true. We rely on the properties of `flagPotentialStraggler` and the logical topology having only a single path to ensure that it eventually becomes true.

- `next: int`
	- A state variable the describes which monitor process to query next from the current process. By repeatedly updating `next` to be $\pred{i}$, we can backtrack until we reach a straggler node.

- `stragglerNode: (int | None)`
	- A state variable that declares the ID of a node known to be a straggler. Initially set to `None` when we know of no such node.
	- This state variable cannot be noisy, in the sense that once it is no longer `None`, it cannot revert to being `None`or a different value.

We also have the following operator for each monitor process:

- `starved(localMetrics: LocalMetrics) -> bool` 
	- When executed on a monitor process $i$ using metrics messaged by monitor $j$ at time $t'$, this operator returns the result of evaluating $L^i(t) < L^j(t')$ at time $t$.
	- Intuitively, this predicate returns true when $i$ is starved for chunks, and has received strictly less than $j$ (note that $L^i(t)$ is non-decreasing, and $t' < t$ by causality, thus the comparison being strict makes sure that if it returns true, $i$ is surely in front of a straggler on the upstream)

- `getLocalMetrics() -> LocalMetrics`
	- Returns the current sampled `LocalMetrics` struct.
	  
- `getMetricsAt(t) -> (LocalMetrics | None)`
	- When called at time $t' \geq t$, it returns the metrics that were sampled at time $t$.
	- When called at a time $t' < t$, this can return `None` or otherwise raise an error as that is not probably intended to happen.

We define the following RPC:

- `probeMonitorProcessRPC(source: int, metrics: LocalMetrics) -> (int | None)`
	- Invoked by a monitor process with ID `source` on another monitor process and either returns an ID or `None`.

The operator `starved` can be defined as follows:
```
DEFINE starved(localMetrics: LocalMetrics)
DO:
	return getMetricsAt(localMetrics.t).L < localMetrics.L;
END DEFINE;
```
>[!NOTE]
>In the above, notice that we are looking up metrics at a particular time. This is why we need synchronized clocks and a history available at each node, since without it, it is possible to have a race condition where between the collection of `localMetrics` and evaluation of `starved`, a burst could have happened from an upstream straggler and the total chunk count recorded in $L$ may have increased significantly, causing the comparison to return false when it is expected to be true.

The probe RPC is defined as follows:
```
RPC probeMonitorProcessRPC(source: int, metrics: LocalMetrics)
DO:
	IF potentialID != None:
		THEN:
			(* A candidate exists, check for if the node is starved *)
			IF starved(metrics) = TRUE:
				THEN:
					(* Starved node, return None *)
					return None;
				ELSE:
					(* We found a straggler node *)
					return MONITOR_ID;
		ELSE:
			(* By properties of `flagPotentialStraggler`, this *)
			(* node MUST itself be a straggler.                *)
			return MONITOR_ID;
	END IF;
END RPC;
```

>[!IMPORTANT]
>Note that it _IS_ possible for `potentialID` to be `None` in the RPC above. While any process that executes `probeMonitorProcessRPC` _MUST_ have already finished `flagPotentialStraggler`, it is not the case that any process that _HANDLES_ the RPC must have executed that procedure as well (indeed, the straggler nodes most likely have not by the time they are probed).

The localization procedure on monitor `i` executes the following:
```
PROCEDURE localzieStraggler()
ASSUME:
    (* We never enter the localization procedure before *)
    (* having finished `flagPotentialStraggler`.        *)
    potentialID := flagPotentialStraggler();

BEGIN WITH:
	next := potentialID;       \* Begin backtracking from potential straggler
	stragglerNode := None;
    stragglerFound := FALSE;

DO:
	(* Get local metrics *)
	localMetrics := LocalMetrics();
    WHILE stragglerFound = FALSE:
	    (* Probe the next monitor *)
        WITH rpcResult = probeMonitorProcessRPC(localMetrics)
        DO:
			IF rpcResult = None:
	            THEN:
		            (* Starved node, go to next parent *)
		            next := Pred(next);
	            ELSE:
					(* We found a straggler node *)
					stragglerFound := True;
					stragglerNode := rpcResult;
			END IF;
        END WITH;
    END WHILE;
END PROCEDURE;
```