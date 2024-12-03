**Paper:** [Large-scale cluster management at Google with Borg](https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/43438.pdf)

**Note:** A subset of Borg was eventually open-sourced as what we know as Kubernetes.

# Introduction

The topic de jour is *Cluster Schedulers*, a system that manages task assignment and process spawning over a set of heterogenous machines. 
In Google's case, the Cluster Scheduler tries to maximize utilization while:
- Prevent tasks from being starved
- Prevent tasks from eating any more resources than they should
- Admission control 
- A bunch of other things of course ...

The main thing about Borg is that it is designed to treat at least two different types of tasks:
- **Latency sensitive, inelastic processes that are short lived.** Also referred to as *Production* or `prod` jobs. 
- **Elastic, batch jobs that run for minutes to days.** Referred also as Non-Production or `nonprod` jobs.

# Borg Design
## Cluster Hierarchy

Borg handles *Cells*, each cell is:
- A cluster of multiple machines (up to 10 thousands ...)
- Have very high performance, high bandwidth networking between each other, making scheduling among them much easier
- Have much higher, WAN-like latency for communicating with other cells.
- Machines are heterogenous even in cell level, meaning that CPU, RAM, disk and type of processors can vary a lot between machines in the same cell.

Borg handles jobs as specified tasks, which could look like the following:
```
job hello_world {
	runtime = {'cell': 'some cell name ...}
	binary = '<path to some binary> ...'
	args = {port='12345', n='100'}
	requiremnts = {
		ram=100 MB
		disk=200 MB
		cpu=0.1
	}
	// Replica count means how any containers to spawn for 
	// this task ...
	replicas=10000
}
```
The language above is referred to as *BCL*.

Borg does not use VMs, it uses containers (mostly for resource efficiency). Containers are harder to isolate from each other compared to VMs, but Borg is for internal use at Google itself, and as such trading the strict security of VMs with the performance and fast spawn of containers is worthwhile.

This however makes scheduling harder, because containers are processes, and the default Linux tools for scheduling resources for processes are extremely hard to fine tune. In contrast, VMs can slice resources reliably among each other with ease.

Having to use containers always also means that binaries are best statically linked so you can just spawn them in-place and self contained, as such, Borg also expects binaries to be statically linked. If this was not the case, then Borg would have to execute pre-requisite tasks before starting a job (essentially something like `apt-get install blah1 blah2 ...`). Having statically linked binaries means that it will not need to do that.

## Borg Jobs

![[Pasted image 20241121123233.png|400]]

A Borg job is a container that remains pending until it gets the resources. A container can also be *rejected* if pending state timeouts or resources are exhausted. 
In the above, each *update* step is a *rolling update*, meaning that it can spawn a new container if needed. Rolling updates makes the system more reliable, since if an update breaks something, the system will not go down.
On the other hand, updates can break compatibility, which is why RPCs are designed to be compatible even if they don't understand some fields (incidentally, this is one of the baked-in features of `gRPC` and Google probably heavily relies on this for making sure rolling updates don't break things completely).

As you can also see above, Borg can at any time kill a process at will. Borg does not bother with VM migration or any such things, it assumes that:
- Processes are fault tolerant and survive crashing failures
- Processes are smart enough to understand `SIGTERM` 

The second condition means that Borg *waits* a little bit before actually pulling the plug on a job after signaling it to terminate in the hope that the application gracefully stops before Borg comes in with a big hammer ...
The reason we give applications this chance is because it might make their future restart easier (maybe they can checkpoint things so they startup faster).

Borg also allows tasks to preempt each other. This is necessary to make sure `nonprod` jobs to do not starve out latency sensitive jobs. One thing that the paper mentions is that at the scale that Borg operates at, preemptions can *cascade*, meaning that a task with a higher priority preempts one that is lower, but then the lower one preempts a bunch of lower lower priority tasks to continue working, and so on ...
This is bad, since it puts a sudden, huge stress on the cluster scheduler to figure out what to do. As such, Borg solves most of this by preventing high priority `prod` tasks from ever preempting each other.

## Tracking Tasks

Each submitted task has a unique ID that is advertised within Borg clusters with a *Borg Name Service*, which is essentially an internal DNS server.
RPC systems are able to use this information to route RPC requests to correct tasks. So for example, an ID can be `50.jfoo.ubar.cc.borg.google.com`, where:
- `50` is a task index
- `jfoo` is a task name
- `ubar` can be a username specifier (either a single user or a group)

Google's *Chubby* file system will store the namespace data and task health info.

## Borg Hierarchy

![[Pasted image 20241121125435.png|500]]

The user-facing part of each Borg cell is the `BorgMaster`:
- It maintains connection with active `BorgLet` instances on each individual machine
- It replicates the most critical global state using a 5-replica Paxos cluster
- It implements the actual cluster scheduler

The lowest part of Borg is the `Borglet`, which is a process that handles task spawning in each machine (so it is the container manager of each machine). It makes no decision by itself, it waits specifically for the master to tell it what to do.

(READ ABOUT THE SCHEDULER AND THE BIN-PACKING ALGORITHM ...)

## Some Evaluations
### Cell Size Effect

Borg uses large cells deliberately. The reason for this is that as we mentioned, cells have high latency of communication between each other. 

![[Pasted image 20241121132416.png|500]]

In the figure above, a set of 15 traces are run over a Borg cluster with varying number of cells (essentially, a cell is repeatedly divided into sub-cells). Each trace is run eleven times and the minimum and maximum completion time, as well as the 90th percentile are extracted. The figure above is plotting the 90th percentile, which shows that tail latency grows substantially when cells are divided, thus larger cells make the system much more predictable.

On the other hand, you shouldn't put everything into a single cell, because of Fault Tolerance, etc.

### Resource Bucket Size

Borg does not set a bucket size for resources, meaning that there is no macro unit for selecting the size of resources. 
This makes the LP that the scheduler has to solve much more difficult, but it makes the system much more resource efficient. To see why, look at the following:

![[Pasted image 20241121133329.png|500]]

In the above, you can see that resource requirements are very well spread, as such using a bucket would group too many things into the same bucket and waste a huge amount of resources. Thus, Borg does not use a bucket. The unit used for scheduling are *milli-cores* for CPU and *bytes* for memory and disk.
