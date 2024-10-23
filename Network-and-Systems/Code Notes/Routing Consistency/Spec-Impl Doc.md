This documents the process used for convincing me/others that [[No Failover Implementation]] follows [[No Failover]] specification.

As for why I am creating yet another documentation for this:
- This isn't just for me, so I'm going to keep things a bit more abstract and clear (or at least try my best!)
- A few things have changed since the last documentation. There is a native OpenFlow integration now, as well as some changes to the overall architecture. This document will also keep notes on the evaluation of the code written for the final paper graphs.
- Here and there, there will also be some mentions of [[Failure Orchestrator]], so do keep an eye out for that as well.

Beyond that, again, the main purpose of this documentation is to isolate those really important parts of the implementation that are supposed to follow the spec directly, and make it easier for me to convince you and myself that I did my homework correctly.

>[!WARNING] A Note About Pseudocodes
>It's hard to separate important lines in pseudocodes without bolding (which looks terrible). And as such, I have established the following rules:
>
>1. Lines that start with a dagger ($\dagger$) **Can Fail**. 
>2. Lines that start with a XOR sign ($\oplus$) correspond to steps that can be delayed and may only finish after all sorts of processes might have interrupted it's start.
>
>All other lines complete unless they explicitly wait or give up to someone else.

# High Level View of The Current Specification

The general architecture of the controller is the same as before, like the following:

- **Routing Controller (RC):** Schedules a DAG of IRs into the network and keeps an eye on the state of each IR, rescheduling any that might have failed.
- **Network Information Base (NIB):** Following the architecture of [[Orion]], the NIB is logically centralized database that keeps the data that all modules need to operate. 
  Unlike Orion, we are not focusing on developing a simple interface for applications to use our controller, and as such, we are not using a general purpose NIB (and lucky we are indeed, since Orion paper basically tells you nothing about how they implemented it!).
- **OpenFlow Controller (OFC):** The main core of the controller. It installs IRs that were scheduled by the RC and receives ACKs for them, and also decides whether or not to suspend certain switches for now.
  The OFC is broken down into 3 separate submodules to make reasoning about it easier, and to also explore more interesting failure scenarios.
  These submodules are:
	  1. **Event Handler:** Handles keepalives received from a switch and decides whether or not we need to suspend and active switch.
	  2. **Worker Pool:** A pool of high performance threads that convert scheduled IRs into OpenFlow messages and send them.
	  3. **Monitoring Server:** The main frontend of communicating with the switches. It registers connected switches and receives keepalives and IR installation ACKs from the switches.

Switches themselves also have internal structures with specific failure patterns, but we won't discuss that yet, we'll leave it for future me to lay it out, once the controller is cleared up. For now, just know that switches have either a *Simple Model* or a *Complex Model*. We are mostly concerned with the Complex Model that makes a switch a series of modules that can fail independently.

I'll be using the final specification versions, the ones for [Complete and Transient Switch Failures](https://github.com/USC-NSL/RoutingConsistency/tree/master/noFailover/5.switchCompleteTransientFailure) and we'll base all of our discussions on the main TLA+ module in it.

While code, if well written, speaks for itself, I don't think specifications do that. So (again!) I'm going to take some time laying our the macros and definitions in the spec. so that we can have a common language going forward, so bare with me for a moment.

## Spec Preliminaries

As with all TLA+ modules, we start by declaring constants throughout the specification. In brief:

### General Constants

| Constant                                 | Semantics                                          |
| ---------------------------------------- | -------------------------------------------------- |
| `SW`                                     | Set of switches. Assumed to be known beforehand.   |
| `SW_SIMPLE_MODEL` and `SW_COMPLEX_MODEL` | Identifier for simple/complex switch model.        |
| `WHICH_SWITCH_MODEL`                     | Maps `SW` to a member of switch model identifiers. |
| `ofc0` and `rc0`                         | Identifiers for OFC and RC threads                 |
| `CONT_MONITOR` and `CONT_EVENT`          | Identifiers for monitoring server and event handler                                                   |

### Complex Switch Model Constants

| Constant                         | Semantics                                               |
| -------------------------------- | ---------------------------------------------- |
| `NIC_ASIC_IN` and `NIC_ASIC_OUT` | NIC input/output buffer identifiers            |
| `OFA_IN` and `OFA_OUT`           | OpenFlow Agent input/output buffer identifiers |
| `INSTALLER`                      | Switch data plane installer                    |

### Switch State Identifiers

| Constant       | Semantics        |
| -------------- | ---------------- |
| `SW_SUSPEND`   | Switch suspended |
| `SW_RUN`       | Switch OK        |
| `SW_RECONCILE` | Switch marked for reconciliation                 |

### IR  State Identifiers:

| Constant    | Semantics                        |
| ----------- | -------------------------------- |
| `IR_NONE`   | IR not scheduled/installed       |
| `IR_SENT`   | IR sent to switch, not yet ACKed |
| `IR_DONE`   | IR sent and ACKed                |
| `IR_UNLOCK` | (For NIB only) IR available for scheduling in worker pool                                 |

### DAG State Identifiers:

| Constant     | Semantics                    |
| ------------ | ---------------------------- |
| `DAG_UNLOCK` | DAG available for scheduling |
| `DAG_NONE`   | DAG not scheduled            |
| `DAG_SUBMIT` | DAG scheduled                |
| `DAG_NEW`    | DAG created/updated          |
| `DAG_STALE`  | DAG needs to be updated                             |

### Controller To Switch Message Type Identifiers

| Constant                         | Semantics                    |
| -------------------------------- | ---------------------------- |
| `INSTALL_FLOW` and `DELETE_FLOW` | Install/Delete a flow entry  |
| `FLOW_STAT_REQ`                  | Request specific flow status |
| `ALL_FLOW`                       | Matches any flow in the switch                             |

### Switch To Controller Message Type Identifiers

| Constant                | Semantics                                              |
| ----------------------- | ------------------------------------------------------ |
| `INSTALLED_SUCCESSFULY` | IR installed                                           |
| `DELETED_SUCCESSFULLY`  | IR deleted                                             |
| `KEEP_ALIVE`            | Keepalive for OpenFlow                                 |
| `FLOW_STAT_REPLY`       | Reply to a `FLOW_STAT_REQ`                             |
| `ENTRY_FOUND`           | Found a matching flow for the one in a `FLOW_STAT_REQ` |
| `NO_ENTRY`              | Found no matching flow for the one in a `FLOW_STAT_REQ`                                                       |

### Spec Inputs

The spec requires certain inputs to work. We won't cover all of them, but the important ones are the following:

| Constant           | Semantics                                                                                  |
| ------------------ | ------------------------------------------------------------------------------------------ |
| `MaxNumIRs`        | Maximum number of IRs to consider during verification                                      |
| `SW_FAIL_ORDERING` | Indicates which switch, in what order and in what way (i.e. transient or partial) can fail |
| `IR2SW`            | A mapping from IR identifiers to switch identifiers                                        |
| `IR2FLOW`          | A mapping from IR identifiers to flow identifiers (flows here are just groups of IRs)      |
| `MaxNumFlows`      | Maximum number flows to consider during verification                                                                                           |

As such, some assumptions are automatically made about the possible values of the constants. The spec rightfully checks them in the beginning to make sure the spec isn't supplied with weird values for these constants. Check lines 179 - 204 to see the assumptions (look for `ASSUME` keyword).

## NIB

The NIB is the glue that holds all modules together, and it appears basically everywhere. We first start by laying out the semantics of the NIB and it's operations, as well as the data structures inside it, and then proceed with module processes an their explanations. The NIB implementation will require a whole different approach later.

### NIB Variables

| Variable Name        | Semantics                                       |
| -------------------- | ----------------------------------------------- |
| `controllerStateNIB` | State of individual processes of the controller |
| `NIBIRStatus`        | State of individual IRs                         |
| `SwSuspensionStatus` | State of switches                               |
| `IRQueueNIB`         | Queue of IRs for OFC worker pool                |
| `SetScheduledIRs`    | Set of scheduled IRs for each switch                                                |

NIB variables are shared amongst ***all*** modules. The NIB obeys the same constraints of [[Orion]], meanings that:
- It is highly available, distributed data store.
- It is NOT durable …

Here, `IRQueueNIB` is probably the most important part of the NIB. This is a buffer of IRs, tagged with a thread identifier. At each point in time, whenever a buffer item is tagged with a thread ID, then that thread in the controller has *locked* the IR, meaning that this scheduled IR is currently being processed by that thread and as such shouldn't be touched until it is freed for whatever reason. 
Because of this, you'll see that the spec sometimes refers to this queue as the "Tagged Buffer". The contents of this buffer need to be addressable for error checking, and as such, items in this buffer are given IDs as well as tags.

### NIB Definitions

The NIB semantics are important, and as such, we try to be a bit more precise here. 
We first define two utilities:
```
rangeSeq(seq) == {seq[i]: i \in DOMAIN seq}
indexInSeq(seq, val) == CHOOSE i \in DOMAIN seq: seq[i] = val
```
The first finds the range of a sequence as a set, and the second returns the index of a given value in a sequence. Since we use `CHOOSE`, in case that value is not in the set, we get an arbitrary value and TLC might complain. If there is multiple of these values, we get a random choice among these members and TLC will NOT complain.

And the definition:
```
removeFromSeq(inSeq, RID) == 
	[j \in 1..(Len(inSeq) - 1) |-> IF j < RID THEN inSeq[j] ELSE inSeq[j+1]]
```
Which remove an element of a given row ID (i.e. index in the array context) from a given sequence.

To find similar elements in the queue, we define the following:
```
isIdenticalElement(itemA, itemB) == 
	\A x \in DOMAIN itemA:  /\ x \in DOMAIN itemB
							/\ itemA[x] = itemB[x]
```
Note that `DOMAIN` can be used for any relation, and as such, the above effectively compares *fields* among the items and sees if their values are equal. For our spec, all these values are either model values (thread tag and IR fields) or just an integer (ID), and as such, only equal to themselves. So this effectively filters out all items that supersede the domain of `itemA` and show the same mapping.

With the above, we define the following basic relations:
```
NoEntryTaggedWith(buff, threadID) == 
	~\E x \in rangeSeq(buff): x.tag = threadID 
FirstUntaggedEntry(buff, num) == 
	~\E x \in DOMAIN buff: /\ buff[x].tag = NO_TAG
                           /\ x < num
```
It's not hard to see that the second definition returns true, if and only if `num` is indeed the first item in `buff` that has the tag `NO_TAG`. In practice, this means that `num` is the index of the first item in the IR queue that is not being processed by any worker in OFC.

With the above, we define basic relations that we will use to define the macros for NIB (again, macros are just atomic steps of a process, and as such, it is really important to show you what atomic steps is the NIB allowed to take).

```
// Get the index of the first element in 'buff' that is tagged with 'threadID' or 
// return an untagged entry to process.
getFirstIRIndexToRead(buff, threadID) == 
	CHOOSE x \in DOMAIN buff: 
		\/ buff[x].tag = threadID
        \/  /\ NoEntryTaggedWith(buff, threadID)
	        /\ FirstUntaggedEntry(buff, x)
            /\ buff[x].tag = NO_TAG

// Get the index of an identical item in 'buff' that is tagged with threadID
getFirstIndexWith(buff, item, threadID) == 
	CHOOSE x \in DOMAIN buff: 
		/\ buff[x].tag = threadID
        /\ isIdenticalElement(buff[x].item, item)

// Is there an identical item in 'buff' that has a unique ID (i.e. not -1) ?
existEquivalentItemWithID(buff, item) == 
	\E x \in rangeSeq(buff): 
		/\ isIdenticalElement(item, x)
        /\ x.id # -1 

// Is there an identical item in 'buff' at all?
existsEquivalentItem(buff, item) == 
	\E x \in rangeSeq(buff): isIdenticalElement(item, x)

// Get set of identical items in 'buff'
setEquivalentItems(buff, item) == 
	{y \in rangeSeq(buff): isIdenticalElement(y, item)} 

// Get the ID of some identical item in 'buff'
getIdOfEquivalentItem(buff, item) == 
	CHOOSE i \in {x.id: x \in setEquivalentItems(buff, item)}:TRUE
```
>[!TODO] About `getFirstIndexWith`
>The definition isn't really reflecting the name, as the item returned may not be the first in the sequence. I don't think this is important, since we only use this to remove items and item removal requires no guarantee other than removing one of the instances, but still, a bit of an unfortunate naming ...

You might ask why we are introducing an ID into NIB. Well, it is possible that certain IRs can get scheduled multiple times and we may end up with duplicate entries in the queue. This isn't avoidable (at least we think that is the case). Putting IDs on these entries, allows us to make sure that we actually detect these in the NIB and ignore them. As such, when an item in the tagged buffer has no ID (i.e. -1), we need to make sure it's actually valid first, and only then assign an ID to it.

Assigning IDs is quite simple, we just assign numbers monotonically (there is a maximum of course for the spec, but just think of it as a really large number for now, but the associated constant is `MAX_IR_COUNT`). We use `increment(irCounter, MAX_IR_COUNT)` to generate these IDs, but you can think of it as just a generic `getNewID` statement.

### Macros

NIB supports the following macros (i.e. these are the atomic steps associated with NIB):

#### `getIdForNewItem`

The following pseudocode correspond to the lines 963 - 971.
```pseudo
\begin{algorithm}
	\caption{getIdForNewItem}
	\begin{algorithmic}
		\Require $buff, item, id$
		\If{existEquivalentItemWithID($buff, item$)}
			\State $id \gets$ getIdOfEquivalentItem($buff, item$)
		\Else
			\State $id \gets$ getNewID()
		\EndIf
	\end{algorithmic}
\end{algorithm}
```

#### `modifiedEnqueue`

The following pseudocode correspond to the lines 973 - 976.
```pseudo
\begin{algorithm}
	\caption{modifiedEnqueue}
	\begin{algorithmic}
		\Require $buff, item$
		\State Append($buff$, [item $\rightarrow item$, id $\rightarrow -1$, tag $\rightarrow$ NO\_TAG])
	\end{algorithmic}
\end{algorithm}
```

This effectively appends an item with empty identifier and tags. Meaning that it has not yet been checked (so ID is -1) as isn't being process (so tagged with `NO_TAG`).

#### `modifiedRead`

The following pseudocode correspond to the lines 978 - 984.
```pseudo
\begin{algorithm}
	\caption{modifiedRead}
	\begin{algorithmic}
	\Require $buff, output, entryIndex$
	\State rowIndex $\gets$ getFirstIrIndexToRead($buff,$ self)
	\State $output$ $\gets buff$[rowIndex].item
	\State getIdForNewItem($buff, output, entryIndex$)
	\State $buff$[rowIndex].tag $\gets$ self
	\State $buff$[rowIndex].id $\gets entryIndex$
	\end{algorithmic}
\end{algorithm}
```

Note that only Worker Pool threads call this, and as such `self` refers to the process identifier of one of these threads. 
Here, `rowIndex` will correspond to the index of either:
- An IR currently locked by this thread, and we return it to resume process
- An untagged IR, which we will now lock

Once `rowIndex` is found, we lock it by tagging it with our thread ID and giving it a new ID to show that this is verified to be processed.

#### `modifiedRemove`

The following pseudocode correspond to the lines 986 - 990.
```pseudo
\begin{algorithm}
	\caption{modifiedRemove}
	\begin{algorithmic}
		\Require $buff, item$
		\State rowRemove $\gets$ getFirstIndexWith($buff, item$, self)
		\State buff $\gets$ removeFromSeq($buff$, rowRemove)
	\end{algorithmic}
\end{algorithm}
```

### Persistency 

It is important to know that when we talk about module or switch failures, we don't of course mean that in the literal sense of a process just stopping and immediately losing it's state. Indeed, as one might expect, "failure" is a valid step for a process, which is enabled in certain key steps of a process and may happen non-deterministically and may resolve with an arbitrary delay (though, since all processes are *fair*, failure resolution cannot be eternally delayed).

Across these failures, certain variables persist, and some of them don't. Since the NIB is the central source of persistence across the controller, we want to take the time and clarify what variables are persistent and which ones are not.

In brief, as long as certain steps do not involve the act of clearing a variable:
- NIB variables are persistent. These being `controllerStateNIB`, `NIBIRStatus`, `SwSuspensionStatus`, `IRQueueNIB` and `SetScheduledIRs`.
- Process variables for both RC and OFC.
- Pretty much no switch variable is persistent by definition. Meaning that a failure step explicitly entails clearing some of them at some point.

It is important to note that while OFC and RC process variables are persistent, they are not necessarily needed to be in implementation. This is because after failure, the module is forced to repeat certain steps, which most likely will re-compute these variables from the information on the NIB and other persistent variables.

To make things as clear as possible, we devote this part to showing you how and why and in what way, each and every variable needs to be persistent. We exclude some helper and auxiliary variables, since they are by definition, totally persistent, but they do not require to be in necessarily in the implementation.

| Variable              | Persistency                                         |
| --------------------- | --------------------------------------------------- |
| `controllerStateNIB`  | NIB variable, Complete Persistence                  |
| `NIBIRStatus`         | NIB variable, Complete Persistence                  |
| `SwSuspensionStatus ` | NIB variable, Complete Persistence                  |
| `IRQueueNIB `         | NIB variable, Complete Persistence                  |
| `SetScheduledIRs `    | NIB variable, Complete Persistence                  |
| `TEEventQueue `       | RC variable, Complete Persistence                   |
| `DAGEventQueue `      | RC variable, Complete Persistence                   |
| `DAGQueue `           | RC variable, Complete Persistence                   |
| `DAGState`            | RC variable, Complete Persistence                   |
| `RCNIBEventQueue`     | RC NIB Event Handler variable, Complete Persistence |


## Processes

The spec is defined in terms of one gigantic algorithm `stateConsistency`. It is not a good idea to just start with the macros and variables, so let's break down individual processes first. Before that, let's quickly breakdown how processes are identified and synchronized. Feel free to just skip this if you are familiar with [[PlusCal]].

All processes are identified with a unique member of a set that contains the identifiers of all processes running within the algorithm. It is in general, good practice to use sets of model values to define these identifiers, as they are:
- Guaranteed to be comparable with each other
- Guaranteed to only equal to themselves 

This is why all those weird constants exist at the beginning of the specification to distinguish threads and processes. A process can always request it's own identifier with the `self` keyword.

Our process identifiers are pairs of model values. For RC and OFC, the <u>first</u> member is `rc0` and `ofc0` respectively. For switches, the <u>second</u> member (for whatever reason not the first) is the switch identifier picked from the `SW` set.

Each process is then executed by taking a series of steps, in the exact order defined by the specification (with the occasional jumps using `goto`), however, TLC decides which process takes a step at any moment by itself, and the algorithm should work under all of the combinations.

Our spec also defines a series of locks, to actually exert some control on which module can proceed first. This has no bearing on the logical run of the algorithm though (i.e. any execution with the locking system, has an equally valid execution without the lock) and it only exists to prevent TLC from taking too long. 
These are defined with a series of macros (i.e. atomic steps) for both the controller modules (`controllerLock`) and switches (`switchLock`). Since our processes are identified with a pair of model values, we can thus just use a pair of model values for signifying who has the lock, as well as a constant `(NO_LOCK, NO_LOCK)` to show that no one actually has the lock.

As such, for switches, we have the following locking macros:
- `switchWaitForLock`: block until controller gives up it's lock and you own the switch lock
- `switchAcquireLock`: block on `switchWaitForLock` and then assign the lock to yourself
- `acquireAndChangeLock(next)`:  block on `switchWaitForLock` and then pass it to the switch identified with `next`.
- `releaseLock(holder)`: release the lock fully from the switch identified with `holder` (remember, the switch has multiple modules). This requires that either the lock be unlocked, or the `holder` switch specifically owning the lock (i.e. `switchLock[2] = holder[2]`).

For controllers:
- `controllerWaitForLockFree:` same as `switchWaitForLock` but for controller modules
- `controllerAcquireLock`: same as `switchAcquireLock` but for controller modules
- `controllerReleaseLock`: wait for the lock and then free it. As such, only controller processes themselves can actually free `controllerLock`.

You'll definitely see these locks being used in all sorts of different ways, but they have no bearing on the actual process steps, and as such, we quite simply ignore discussing them further.

The algorithm is made up of the following processes (look for `fair process` keywords).

### Routing Controller (RC)

RC schedules DAGs of IRs, and awaits confirmation on their states. It also handles DAG migration in case a failure occurs that wipes out certain previously installed IRs from the network.

#### Variables

The following are the variables maintained locally in the RC:

| Variable Name          | Variable Description                                                                    |
| ---------------------- | --------------------------------------------------------------------------------------- |
| `TEEventQueue`         | Queue for topology change events                                                        |
| `DAGEventQueue`        | Queue for DAG change events                                                             |
| `DAGState`             | State of each DAG. Contains members of [[#DAG State Identifiers]]                       |
| `RCNIBEventQueue`      | Queue of NIB events published for RC                                                    |
| `RCIRStatus`           | A local copy of `IRStatus` from [[#NIB Variables]]                                      |
| `RCSwSuspensionStatus` | A local copy of `SwSuspensionStatus` from [[#NIB Variables]]                            |
| `staleDagId`           | Holds the ID of the DAG that will be considered stale when a new DAG is to be submitted |
| `downSwSet`            | Set of switches that are currently down                                                                                        |

#### Macros

##### `getSetRemovableIRs`

This macro computes the set of IRs that the current DAG requires and whose corresponding switch is still in the topology. A removable must have the following conditions:
- It corresponds to an active switch
- It is not part of the current DAG
- It's current state is not `IR_NONE` ***or*** has been added to scheduled set of IRs

This can be expressed as follows:
```
getSetRemovableIRs(activeSwSet, DAGVertices) == 
	{x \in 1..MaxNumIRs}: /\ \/ RCIRstatus[x] # IR_NONE
						     \/ x \in SetScheduledIRs[ir2sw[x]]
						  /\ x \notin DAGVertices
						  /\ ir2sw[x] \in activeSwSet}
```

##### `getSetIRsForSwitchInDAG`

This macro just returns the set of IRs associated with a single switch in the current DAG:
```
getSetIRsForSwitchInDAG(swID, DAGVertices) == {x \in DAGVertices: ir2sw[x] = swID}
```
##### `isDependencySatisfied`

Returns true if and only if all dependencies of a given IR have been satisfied in a given DAG:
```
isDependencySatisfied(DAG, ir) == 
	~\E y \in DAG.v: /\ <<y, ir>> \in DAG.e
		             /\ RCIRStatus[y] # IR_DONE
```
##### `getSetIRsCanBeScheduledNext`

This macro finds the set of schedulable IRs. These IRs must have all the following conditions:
- The IR status is `IR_NONE`
- The dependencies of the IR have been satisfied
- The destination switch of the IR is active
- IR has not been scheduled yet
- All switches corresponding to the dependencies of this IR are already active

```
getSetIRsCanBeScheduledNext(DAG) == 
	{x \in DAG.v: /\ RCIRStatus[x] = IR_NONE
                  /\ isDependencySatisfied(DAG, x)
                  /\ RCSwSuspensionStatus[ir2sw[x]] = SW_RUN
                  /\ x \notin SetScheduledIRs[ir2sw[x]]
                  /\ ~\E p \in Paths(Cardinality(DAG.v), DAG.e): 
	                  /\ p[Len(p)] = x
                      /\ RCSwSuspensionStatus[ir2sw[p[1]]] # SW_RUN}
```
Here, `Paths` just returns the set of all paths of length at most `Cardinality(DAG.v)` that end in a given edge's end.

##### `DAGIsDone`

Returns true if all IRs in a DAG have been installed.
```
DAGIsDone(DAG) == ~\E y \in DAG.v: RCIRStatus[CID][y] # IR_DONE
```

#### RC Processes

The following are the processes making up RC:

| Process Name                   | Process Description                                                |
| ------------------------------ | ------------------------------------------------------------------ |
| `rcNibEventHandler `           | Consumes events published for RC from the NIB.                     |
| `controllerTrafficEngineering` | Consumes topology change events and re-computes/un-schedules DAGs. |
| `controllerBossSequencer`      | Schedules DAGs of IRs                                              |
| `controllerSequencer`          | Pool of workers that schedule IRs within a DAG scheduled by the boss.                                                                   |

##### `rcNibEventHandler `

This process will consume NIB events for RC. These NIB events are setup by other modules for RC to reschedule certain IRs or to start redoing things if needed. These events are generated when:
1. Whenever the state of an IR has been changed, due to whatever instruction may have been pushed or whatever failure may have been detected. We call these `IR_MOD` events.
2. A switch undergoes a failure and is suspended. This generates a `TOPO_MOD` event where the RC is informed by the NIB to simply ignore this switch and all IRs associated with it, until the Monitoring Server and Event Handler decide that all is well and the switch has resumed operation.

###### `TOPO_MOD` Event Message

| Field Identifier | Field Semantics                 |
| ---------------- | ------------------------------- |
| `type`           | The unique value for `TOPO_MOD` |
| `sw`             | A switch identifier             |
| `state`          | A member in [[#Switch State Identifiers]]                                |

###### `IR_MOD` Event Message 

| Field Identifier | Field Semantics               |
| ---------------- | ----------------------------- |
| `type`           | The unique value for `IR_MOD` |
| `ir`             | An IR object                  |
| `state`          | A member in [[#IR State Identifiers]]                              |

To handle these, we simply wait on the queue `RCNIBEventQueue` and pop events and process them. 

###### Pseudocode

The following pseudocode correspond to lines 1485 - 1504.

```pseudo
\begin{algorithm}
\caption{RCNIBEventHandler}
	\begin{algorithmic}
		\Require Len(RCNIBEventQueue) > 0
		\While{True}
			\State $event \gets$ Head(RCNIBEventQueue)
			\IF{$event.type = $ TOPO\_MOD}
				\If{$event.state \neq$ RCSwSuspensionStatus[$event.sw$]}
					\State RCSwSuspensionStatus[$event.sw$] $\gets event.state$
					\State Append(TEEventQueue, $event$)
					\If{$event.state = $ SW\_RUN}
						\State \textnormal{Un-schedule all IRs for switch $event.sw$}
					\EndIf
				\EndIf
			\ElsIf{$event.type = $ IR\_MOD}
				\If{RCIRStatus[$event.IR$] $\neq event.state$}
					\State RCIRStatus[$event.IR$] $= event.state$
					\If{$event.state \in$ \{IR\_SENT, IR\_DONE\}}
						\State Remove $event.IR$ from SetScheduledIRs[ir2sw[$event.IR$]]
					\EndIf
				\EndIf
			\EndIf
			\State Pop(RCNIBEventQueue)
		\EndWhile
	\end{algorithmic}
\end{algorithm}
```

##### `controllerTrafficEngineering`

This process will consume topology change events setup by the NIB. The `rcNibEventHandler` process will setup these events in `TEEventQueue` and this process will then consume them. The process should do the following in sequence:

1. Consume a topology change event from `TEEventQueue` and determine the set of suspended switches.
2. Determine the next DAG according to set of failed switches
	 - If the DAG has not changed, go to step 1
	 - If the DAG has changed, mark the previous DAG as `DAG_STALE` by generating an event in `DAG_EVENT_QUEUE`. You may have to give IDs to DAGs for this step, so keep a `getDAGID()` function somewhere. Afterwards, find the set of unnecessary IRs (those that are not in the new DAG). 
	   The process will now wait for RC Boss Sequencer to process the `DAG_STALE` event. Once the state of the previous DAG has been changed to `DAG_NONE`, we resume and submit the DAG.
3. Once the previous DAG is unscheduled, calculate the set of removable IRs and proceed to submit the DAG by changing it's state to `DAG_SUBMIT` and appending it to `DAGEventQueue`.

All that is the left, is to delete the unnecessary IRs calculated in step 2. To do this, we need to schedule `DELETE_FLOW` IRs for these appropriate IRs. To do this, we first use a function `getDeleteIRID()` to generate and ID for these IRs, and then we schedule them by:
- Choosing an IR from the set of unnecessary IRs calculated in step 2 above. Call it `currIR`
- Setting their `RCIRstatus` entry and `NIBIRStatus` entry to `IR_NONE`.
- Adjust IR type to be `DELETE_FLOW`
- Set new IR destination to be the same as `currIR`
- Set this IR to be the predecessor of all of the destination switch IRs in the current DAG

>[!NOTE] Getting Initial Next DAG
>When a switch in a topology goes down, a non-trivial algorithm should run on the controller to determine the appropriate DAG to use for the next round. In [[Orion]] for example, that algorithm is said to route around a downed switch when there isn't *too many* down switches, or just do nothing when there is too many down switches.
>We will not put this algorithm in our specification, it makes things way too complicated. Instead, a constant `TOPO_DAG_MAPPING` is used to map the set of downed switches to a plausible DAG that we will then use.

###### New DAG Event Message

| Field Identifier | Field Semantics      |
| ---------------- | -------------------- |
| `type`           | `NEW_DAG` identifier |
| `dag_obj`        | A record of form $<dag \rightarrow \textnormal{DAG Object}, id \rightarrow \textnormal{DAG ID}>$                     |

###### Stale DAG Event Message

| Field Identifier | Field Semantics        |
| ---------------- | ---------------------- |
| `type`           | `STALE_DAG` identifier |
| `id`             | ID of the staled DAG                       |

###### Process TE Event
The following corresponds to lines 1522 - 1534.
```pseudo
\begin{algorithm}
\caption{RCProcessTEEvent}
	\begin{algorithmic}
		\Require Len(TEEventQueue) > 0
		\While{Enabled}
			\State $topoEvent \gets$ Head(TEEventQueue)
			\If{$topoEvent.state =$ SW\_SUSPEND}
				\State \textnormal{Add $topoEvent.sw$ to current set of DOWN switches}
			\Else
				\State \textnormal{Remove $topoEvent.sw$ from the current set of DOWN switches}
			\Endif
			\State Pop(TEEventQueue)
		\EndWhile
	\end{algorithmic}
\end{algorithm}
```

###### Send Stale DAG Notification
The following corresponds to lines 1547 - 1549.
```pseudo
\begin{algorithm}
\caption{SendStaleDAGNotif}
	\begin{algorithmic}
		\Require $staleDagId$ from \textbf{"Compute DAG"}
		\State $dagStaleEvent = $ [type $\rightarrow$ DAG\_STALE, id $\rightarrow staleDagId$]
		\State Append(DAGEventQueue, $dagStaleEvent$)
	\end{algorithmic}
\end{algorithm}
```

###### Wait For Stale DAG Removal
The following corresponds to lines 1551 - 1556.
```pseudo
\begin{algorithm}
\caption{WaitForStaleDAGRemoval}
	\begin{algorithmic}
		\Require DAGState[$staleDagId$] = DAG\_NONE \textnormal{and $nextDAG$ from \textbf{"Compute DAG"}}
		\State $staleDagId \gets nextDAG$.id
		\State $staleDag \gets nextDag$.dag
		\State $setRemovableIRs \gets$ getSetRemovableIRs(SW \textbackslash downSwSet, nextDAG.v)
	\end{algorithmic}
\end{algorithm}
```

###### Compute DAG
The following corresponds to lines 1536 - 1562 and contains previous algorithms.
```pseudo
\begin{algorithm}
\caption{ComputeDAG}
	\begin{algorithmic}
		\State nextDAG $\gets$ [id $\rightarrow$ getDAGID(), dag $\rightarrow$ TOPO\_DAG\_MAPPING[$downSwSet$]]
		\If{$staleDag =$ nextDAG.dag}
			\State \textnormal{Nothing to do! Go back and wait for a TE event}
		\Else
			\State DAGState[$staleDagId$] $\gets$ DAG\_STALE
			\State $\oplus$ \textbf{"Send Stale DAG Notification"}
			\State $\oplus$ \textbf{"Wait For Stale DAG Removal"}
		\EndIf
	\end{algorithmic}
\end{algorithm}
```

###### Add Dependency Edges
This process adds dependencies to a DAG that correspond to the removal of the IRs from the previous DAG.
This corresponds to lines 1579 - 1587.
```pseudo
\begin{algorithm}
\caption{AddDependencyEdges}
	\begin{algorithmic}
		\Require $setIRsInDAG$ and $dependencyIR$ from \textbf{"Remove Unnecessary IRs"}
		\State $currentIR \gets$ RandomPick($setIRsInDAG$)
			\State Add $<dependencyIR, currentIR>$ to $nextDAG.dag.e$ 
	\end{algorithmic}
\end{algorithm}
```

###### Remove Unnecessary IRs
This corresponds to lines 1564 - 1589.
```pseudo
\begin{algorithm}
\caption{RemoveUnnecessaryIRs}
	\begin{algorithmic}
		\Require $setRemovableIRs$ from \textbf{"Compute DAG"}
		\State $chosenIR \gets$ RandomPick($setRemovableIRs$)
		\State $dependencyIRID \gets$ getDeleteIRID()
		\State AddEntry(RCIRStatus, $dependencyIRID \rightarrow $IR\_NONE)
		\State AddEntry(NIBIRStatus, $dependencyIRID \rightarrow $IR\_NONE)
		\State \textnormal{Add $dependencyIRID$ to $nextDAG.dag.v$}
		\State $setIRsInDAG \gets$ getSetIRsForSwitchInDAG(ir2sw[$chosenIR$], $nextDAG.v$)
		\State $\oplus$ \textbf{"Add Dependency Edges"}
	\end{algorithmic}
\end{algorithm}
```

###### Submit New DAG
This corresponds to lines 1591 - 1594.
```pseudo
\begin{algorithm}
\caption{Submit New DAG}
	\begin{algorithmic}
		\State DAGState[nextDAG.id] $\gets$ DAG\_SUBMIT
		\State $dagEvent \gets $[type $\rightarrow$ DAG\_NEW, dag\_obj $\rightarrow nextDAG$]
		\State Append(DAGEventQueue, $dagEvent$)
	\end{algorithmic}
\end{algorithm}
```

###### Controller Traffic Engineering
This corresponds to lines 1510 - 1597.
```pseudo
\begin{algorithm}
\caption{ControllerTrafficEngineering}
	\begin{algorithmic}
		\Require Len(TEEventQueue) > 0
		\State $\oplus$ \textbf{"Process TE Event"}
		\State $\oplus$ \textbf{"Compute DAG"}
		\State $\oplus$ \textbf{"Remove Unnecessary IRs"}
		\State $\oplus$ \textbf{"Submit New DAG"}
	\end{algorithmic}
\end{algorithm}
```

##### `controllerBossSequencer`

The Boss Sequencer consumes events in  `DAGEventQueue`, which are either a [[#New DAG Event Message]] or [[#Stale DAG Event Message]].

>[!FAIL] TODO
>Ask Pooria! I have no idea why this part is fault tolerant!! 

>[!UPDATE]
>Turns out it is not!
>We need to add a few things to this part of the spec!


##### `controllerSequencer`

### OpenFlow Controller (OFC)

The OFC is made of 3 distinct processes, each acting as a separate module of the specification. These are:
- **Monitoring Server:** Connects to the switches and performs handshakes as well as collecting events from them (or generating it).
- **Event Handler:** Handles switch specific events that were gathered in the Monitoring Server.
- **Worker Pool:** A pool of workers, each processing a scheduled IR to completion.

The main job of OFC is to schedule the IRs sent to it by RC. These IRs take three forms:
- `INSTALL_FLOW` where the RC instructs the OFC to send an flow to a switch and wait for acknowledgements.
- `DELETE_FLOW` where the RC instructs the OFC to delete a specific IR in the switch.
- `FLOW_STAT_REQ` where the RC instructs the OFC to collect the current state of a flow.

#### Variables

##### Worker-Specific Variables

| Variable Name  | Variable Description                            |
| -------------- | ----------------------------------------------- |
| `nextIRToSend` | The IR that the thread will not attempt to send |
| `entryIndex`   | Unique NIB identifier for `nextIRToSend`        |

##### Event Handler Variables

| Variable Name     | Variable Description                                         |
| ----------------- | ------------------------------------------------------------ |
| `monitoringEvent` | Current monitoring event received from the monitoring server |

#### OFC Processes

##### `ControllerThread`

This process is the main worker pool. It converts the IRs scheduled by RC into OpenFlow messages and sends them to the specific switches.

One important thing for this process is that since there might be more than one thread trying to access the same IR, then it is quite possible that we can have a race condition between these two threads. As such, a locking mechanism is used to prevent that, but since during implementation we will solve this in another way, we won't detail it too much an simply write that the worker will tag an IR with it's ID.