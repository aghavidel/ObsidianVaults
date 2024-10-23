# VM Fault Tolerance

## Recap Of RAMCloud/GFS

- Single coordinator like GFS
- Data hierarchy: 
	- Table -> Tablet -> Object
- RAMCloud provides strong consistency
- Replication is done in log-entry level, when an update is pushed to the log, it is replicated across the cloud, and if the master fails, the logs will recover the updates.

It is probably good to recap what we learned with GFS and RAMCould. The main difference between the two is **what** they replicate:
- RAMCloud replicates *states*, which means that the most recent entry of a log contains the most recent value, and as such, crash recovery is much faster.
- For GFS, *requests* are replicated to the log, which means that if a crash happens, you must REPLAY the whole log to get the correct values.

In general, request replication is much easier and more general, but makes crash recovery slower. This particular method of replication (i.e. a single master serializing requests and assigning a single primary for keeping data). This is called the **Primary-Backup Replication**, where:
- One replica is a primary, serving writes first, and others are backups.
- Primary orders client requests on data-path
- Only a single primary exists at any moment

>[!IMPORTANT] Key Point
>For Primary-Backup replication, we MUST consistently keep an eye on who is the master and who is the primary.

>[!NOTE] Side Note: Asynchronous Replication
>There is another method of replication, Asynchronous Replication. This means that when a client sends a request to the primary:
>1. Primary gets client operation
>2. Primary executes the operation in place and ACKs the client
>3. Primary orders client requests
>4. Primary writes received client requests to backup asynchronously
>5. Backup goes live if primary dies
>
>This can create inconsistencies, especially if the primary dies without being able to update the backup.

This replication mechanism while simple, is not generic and depends on the applications. Today, we will discuss a more general replication method:

## State Machine Replication

The idea is that each replica is a *state machine*, as in, a set of key-value pairs, and a log of operations that transition the state from some initial value to its current value.
- **We assume:** state machines are *deterministic*, so if provided the same set of inputs and operations, they end up in the same state.
- **We design:** to make sure that we can provide *all-or-nothing replication*, which means that any operation is either replicated to all replicas, or none of them.

A state machine can be replicated synchronously or asynchronously. We consider synchronous replication:
- A primary receives an operation
- It logs the operation locally
- It replicates the operation to all other replicas 
- It executes the operation on its state machine and returns the result

We will now discuss a practical implementation of the State Machine Replication, the Fault Tolerant VM system from VMWare.

## VM-FT

The goals are:
1. Replicate the whole VM
2. Make replication completely transparent to the user and application
3. Highly Available

So our first main question, **replicate WHAT?**
Again, we can either replicate state change or the state value, and since we are replicating an entire VM, then replicating state changes make much more sense, since VM states can be very large.
For the case of VMs however, they receive inputs as operations as well (interrupts, keyboard/mouse inputs, network egress/ingress packets); we must replicate these as well.

VM-FT is designed with at least 2 virtual machines:
- A primary and backup VM, which could run on different bare metal. The backup runs exactly like primary with a small lag
- The two VMs are synchronized with a logging channel between them
- A shared disk MUST exist between the two VMs, connected with fiber in VMWare's case

All outputs from the backup are ignored while the primary is active. A VM is said to "Go Live", when it closes the logging channel and externalizes all of its outputs to the user like normal.

There are 3 challenges to VM-FT:
1. Make the backup look like an exact replication of the primary
2. Make the system behave like a single machine
3. Avoid any possibilities of two primaries working at one (split-brain problem)

### Handling Non-Determinism

So one problem is that if we want to replicate using a log, we need to make sure the state machine is deterministic, but the problem is that there are many instructions that are NOT deterministic by nature (for example, reading CPU clock). These instructions must be replicated as values in the log, without having to be executed at the backup like other instructions.

Logging is done by the hypervisor in the primary, and all log entries generated here are streamed to the backup VM. The backup executes the log and as such lags behind the primary a little bit, but that is fine.

>[!IMPORTANT] Output Requirement
>The primary must never externalize output BEFORE the backup has ACKed that it has received the log entry that signifies the output.
>The reason this is needed is that if the ACK and the primary both fail, then the instruction will be lost to the backup, and so it forgets to execute all in-flight instructions.

>[!FAQ] Duplicate Executions
>It is possible to have duplicate outputs. If primary externalizes output but immediately dies, then backup would have no way to know whether or not the primary did manage to externalize the output or not. As such, it must repeat this again.
>This if fine generally, since NOT having an output is generally much better than having duplicates (in the network for example, TCP can handle that). We can also do the same when it comes to disk writes.

#### Primary-to-Backup Failover

Since non-deterministic instructions were not actually executed local to backup, it is possible that once the backup takes over, it can execute very differently. But that is okay, as long as the backup generates the exact same output compared to the case where the primary just executed by itself.

>[!NOTE] What Does Above Mean?
>If primary asks the OS for a random number that is truly non-deterministic, it might get value $A$, but if it dies before being able to log the value of $A$ to backup, then when backup takes over, it will execute this same instruction and possibly get a different value of $B$.
>
>Now, the application might react differently depending on the value of $A$ and $B$, but that is dependent on the application, so it is ok.

### Avoiding Split-Brain

If the logging channel fails, the two VMs will end in two different network partition. The channel keeps heartbeats between the two VMs using UDP packets, and in the event that too many requests remain unanswered, the VMs will decided **independently** that the channel is broken. 
- When a primary is chosen, it uses a compare-and-swap operation on the shared disk to make sure that only a single primary exists. If the compare-and-swap fails, the machine halts.
- If the primary gets promoted to the primary, then it will advertise the same MAC address as the previous primary and continues execution.
- If the previous primary did in fact live, it can still receive input, but since it is waiting on the ACKs from a non-existent backup, it will block and never externalize any output.