This code details the implementation of the scenario where no failover protocol is used in the implementation of the control plane.

Before reading this, you MUST have seen the [[Control Plane NSDI Paper]] before, lest some much needed context will be missing.

The repository of the code can be found [here](https://github.com/USC-NSL/RoutingConsistency/blob/master/noFailover/3.bothTransientFailures/verified.tla), through it is not public to my knowledge.

## Modules

### Switches

#### Definitions

Recall that each switch in the spec, had two models that could be used:
- **Simple Switch:** Just receives an IR, installs it and immediately sends an ACK to the control plane.
- **Complex Switch:** Had internal modules:
	- **NIC/ASIC** for the interface to the control plane.
	- **TCAM** for the buffer used for ACKs and keep-alive messages.
	- **Installer** the hardware that installs the OFA output on the data plane.
	- **OFA** the [[OpenFlow]] agent.

The spec allows one to use any of these models on each of the switches given, so we would have:
```
CONSTANTS 
	SW,                   \* Set of switch entities
	SW_SIMPLE_MODEL,      \* Identifier for the simple model
	SW_COMPLEX_MODEL,     \* Identifier for the complex model
	WHICH_SWITCH_MODEL,   \* Map from SW to {SW_SIMPLE_MODEL, SW_COMPLEX_MODEL}
```
To make this more clear, one model parameter set to assign can be:
```
SW = [model value] {s0, s1}
WHICH_SWITCH_MODEL = [s0 |-> SW_COMPLEX_MODEL, s1 |-> SW_COMPLEX_MODEL]
```
Now, the spec will model the entire controller as one big process, running multiple smaller processes. Each process may run it's own set of threads, which will be separated from ones in another module using a unique, constant identifier.

For example, threads in the OFC, will be identified with the identifier `ofc0`, and threads in the RE, will be identified with `rc0` (why RE is instead RC here, I should probably ask Pooria at some point). In the code, these constant identifiers are referred to as `CID`.

Given these, and identifiers for each submodule in a module (consider these as just model values), we will be able to define the set of processes for each component. For example, the set of switch processes will be:
```
SwProcSet = ({NIC_ASIC_IN} \X SW) \cup 
			({NIC_ASIC_OUT} \X SW) \cup
			({OFA_IN} \X SW) \cup 
			({OFA_OUT} \X SW) \cup
			({INSTALLER} \X SW)
```
To this, the set of failure detection and failure resolution processes are also added in the form of `({SW_FAILURE_PROC} \X SW) \cup ({SW_RESOLVE_PROC} \X SW)`.

Throughout all of the spec, failure is denoted simply as a binary status in `{Failed, NotFailed}`. Initially, all switches and components are OK, so:
```
switchStatus = [x \in SW |-> [
	cpu |-> NotFailed,
	nicAsic |-> NotFailed,
	ofa |-> NotFailed,
	installer |-> NotFailed
]]
```
##### Internal Modules

Internal modules of course, need buffers for communication:
```
NicAsic2OfaBuff = [x \in SW |-> <<>>],
Ofa2NicAsicBuff = [x \in SW |-> <<>>],
Installer2OfaBuff = [x \in SW |-> <<>>],
Ofa2InstallerBuff = [x \in SW |-> <<>>],
```
The TCAM also requires it's own buffer (since well, it's literarily a buffer):
```
TCAM = [x \in SW |-> <<>>],
```
We also introduce:
```
controlMsgCounter = [x \in SW |-> 0],
```
This essentially serves as a timestamp counter for messages received from the switch. It is important during event handling for us to look at the most *recent* event. It also helps debugging.

##### Switch and IR States

Similar to how it is discussed in the paper, here we also assign states to switches and IRs. This differs a bit compared to what was presented in the paper though.

| State Identifier | State Semantics                          |
| ---------------- | ---------------------------------------- |
| `IR_DONE`        | IR is finished installing                |
| `IR_NONE`        | IR has never been scheduled      |
| `IR_PENDING`     | IR is scheduled, pending installation in the switch |
| `IR_SENT`        | IR has been sent to the switch, waiting for switch ACK                                         |

##### Operational Identifiers

For internal switch modules, their status within the code may be identified as a binary variable in `{FAILED, NOT_FAILED}`.
```
swCanReceivedPackets(sw) == switchStatus[sw].nicAsic = NotFailed
swCanInstallIRs(sw) == /\ switchStatus[sw].cpu = NotFailed
					   /\ switchStatus[sw].installer = NotFailed
swOFACanProcessIRs(sw) == /\ switchStatus[sw].cpu = NotFailed
						  /\ switchStatus[sw].ofa = NotFailed
```

##### Failure Recovery

For failure recovery, a set of macros are defined to get all failed/running modules in a switch. However, note that as long as the controller does not suffer from failures:

- The controller can always detect NIC/ASIC failures by itself by checking for a timeout in the TCP connection with the switches.
- If the installer or OFA fail, failure can be detected if and only if the CPU is alive.
- CPU failure can be detected, but it leaves the state of other modules (excluding the NIC/ASIC) unspecified.

In brief, if the CPU has not failed, it is possible to detect any failure within the switch. If the CPU has failed, only NIC/ASIC failure and CPU failure may be reported. Therefore, we cannot react to OFA failure here, even if it actually happens.

Thus, we have:
```
returnSwitchElementsNotFailed(sw) == 
	{x \in DOMAIN switchStatus[sw]: switchStatus[sw][x] = NotFailed}
	
returnSwitchFailedElements(sw) == 
	{x \in DOMAIN switchStatus[sw]: /\ switchStatus[sw][x] = Failed
									/\ \/ switchStatus[sw].cpu = NotFailed
									   \/ x \notin {"ofa", "installer"}}
```
>[!TODO]
>Check lines 357 - 361

The installer is of particular importance, therefore:
```
getInstallerStatus(stat) == IF stat = NotFailed
								THEN INSTALLER_UP
								ELSE INSTALLER_DOWN
```
The semantics of `INSTALLER_UP` and `INSTALLER_DOWN` are defined in [[#Event Handler]].

The following definitions will also be used to define macros later:
```
installerInStartingMode(swID) == pc[<<INSTALLER, swID>>] = "SwitchInstallerProc"

ofaStartingMode(swID) == /\ pc[<<OFA_IN, swID>>] = "SwitchOfaProcIn"
						 /\ pc[<<OFA_OUT, swID>>] = "SwitchOfaProcOut"

nicAsicStartingMode(swID) == /\ pc[<<NIC_ASIC_IN, swID>>] = "SwitchRcvPacket"
							 /\ pc[<<NIC_ASIC_OUT, swID>>] = "SwitchFromOFAPacket"
```
These definitions, enable us to detect whether or not certain processes are running, which we can use to enable resolution/reconciliation when required.

#### Macros

Switch macros are defined over all processes in `SwProcSet`, which is:
```
SwProcSet = (
	(({NIC_ASIC_IN} \X SW)) \cup 
	(({NIC_ASIC_OUT} \X SW)) \cup 
	(({OFA_IN} \X SW)) \cup 
	(({OFA_OUT} \X SW)) \cup 
	(({INSTALLER} \X SW)) \cup 
	(({SW_FAILURE_PROC} \X SW)) \cup 
	(({SW_RESOLVE_PROC} \X SW))
)
```
Thus, `self` is a tuple where the first element is one of `{NIC_ASIC_IN, NIC_ASIC_OUT, OFA_IN, OFA_OUT, INSTALLER, SW_FAILURE_PROC, SW_RESOLVE_PROC}`, and the second element is the switch identifier.

Switch macros deal with failures and resolutions. Naturally:
- Failure awaits no other process, meaning that a failure in the switch may occur at any moment
- Failure resolution, naturally, awaits it's corresponding process

Failure macros are:
- `nicAsicFailure()`
	- Atomically forces a NIC/ASIC failure in a switch
	- Enabled when `switchStatus[self[2]].nicAsic = NotFailed`
	- It clears `controller2Switch[self[2]]`
	- The controller notices this, and a `NIC_ASIC_DOWN` event is generated in OFC
- `resolveNicAsicFailure()`
	- Atomically resolves NIC/ASIC failure
	- Awaits `nicAsicStartingMode(self[2])`
	- Enabled when `switchStatus[self[2]].nicAsic = Failed`
	- Sends a status resolution message to the controller. 
		1. If OFA and Installer are OK, then send a `KEEP_ALIVE` with status `INSTALLER_UP`
		2. If OFA is failed, then send `OFA_DOWN`
		3. If OFA is OK, but Installer has failed, send a `KEEP_ALIVE` with status `INSTALLER_DOWN`
- `cpuFailure()`
	- Atomically forces a CPU failure in a switch
	- Sets the status of OFA, Installer and CPU to `Failed`
	- Clears all buffers, which are:
		1. `NicAsic2OfaBuff`
		2. `Ofa2InstallerBuff`
		3. `Installer2OfaBuff`
		4. `Ofa2NicAsicBuff`
	- If NIC/ASIC is OK, send `OFA_DOWN` message. If not, then `nicAsicFailure` must have already sent a `NIC_ASIC_DOWN` message.
- `resolveCpuFailure()`
	- Atomically resolves CPU failure
	- Awaits `ofaStartingMode(self[2])` and `installerInStartingMode(self[2])`
	- Enabled when `switchStatus[self[2]].cpu = Failed`
	- Sets CPU status to `NotFailed`. Naturally, OFA (i.e. some software running with the OS that has a functioning CPU) will also recover now, so we also set it to `NotFailed`.
	- If NIC/ASIC is OK, send `KEEP_ALIVE` with the current status of the Installer to the controller. If not, then `nicAsicFailure` must have already sent a `NIC_ASIC_DOWN` message. 
- `ofaFailure()`
	- Atomically forces OFA failure
	- Enabled when OFA and CPU are *both* `NotFailed` (the CPU may seem unnecessary, but consider that if the CPU were to fail, the OFA would also go with it, thus considering the case where the OFA fails while the CPU has already failed, is vacuous)
	- Set OFA status to `Failed`
	- If NIC/ASIC is OK, send `OFA_DOWN` message
- `resolveOfaFailure()`
	- Atomically resolves OFA failure
	- Awaits `ofaStringMode(self[2])`
	- Enabled when CPU is `NotFailed` and OFA is `Failed`
	- Set OFA status to `NotFailed`
	- If NIC/ASIC is OK, send `KEEP_ALIVE` with the current Installer status
- `installerFailure()`
	- Atomically forces Installer failure
	- Same as OFA, it is only enabled when both the CPU and the Installer are in `NotFailed`
	- Sets Installer to `Failed`
	- If NIC/ASIC is OK, send `KEEP_ALIVE` with `INSTALLER_DOWN`
- `resolveInstallerFailure()`
	- Atomically resolves installer failure
	- Awaits `installerStringMode(self[2])`
	- Enabled when CPU is `NotFailed` and Installer is `Failed`
	- Sets Installer to `NotFailed`
	- If NIC/ASIC is OK, sends `KEEP_ALIVE` with `INSTALLER_UP`

#### Procedures

##### Flow Installation

- Installing flow entries in OFA
```pseudo
ofaInstallFlowEntry:
	INPUT: "ofaInIR" The received IR
	
	SendRcvConfirmationToController:
		IF "OFA can process IRs"
			THEN "Append RECEIVED_SUCCESSFULLY to Ofa2NicAsicBuff"
			ELSE return
	SwitchOFAInsert2InstallerBuff:
		IF "OFA can process IRs"
			THEN "Append received IR ofaInIR to Ofa2InstallerBuff"
			ELSE return
```
- Respond to reconciliation request:
  Reconciliation requests are just there to inform the controller about any surviving state in the switch, which includes cached IRs that the controller isn't sure whether or not they survived or not, or if they were ever sent in the first place.
```pseudo
ofaProcessReconcileRequest:
	INPUT: "ofaReconcileIR" Some cahced IR to send to the controller
	
	OfaLookAtInstalledCache:
		IF "OFA cannot process IRs"
			THEN return
		ELSE IF "ofaReconcileIR was previously cached in OFA"
			THEN "Append RECONCILIATION_RESPONSE to Ofa2NicAsicBuff with status
			INSTALLED_SUCCESSFULLY" 
			// Note: The above is equivalent to informing the controller that
			// the IR is installed
			return
		
	OfaLookAtReceivedCache:
		IF "OFA cannot process IRs"
			THEN return
		ELSE IF "ofaReconcileIR was previously cached in OFA"
			THEN "Append RECONCILIATION_RESPONSE to Ofa2NicAsicBuff with status
			RECEIVED_SUCCESSFULLY" 
			// Note: The above is equivalent to informing the controller that
			// the IR is received
			return
			
	OfaNoAvailableStatus:
		IF "OFA can process IRs"
			THEN "Append RECONCILIATION_RESPONSE to Ofa2NicAsicBuff with status
			STATUS_NONE" 
			// Note: The above is equivalent to informing the controller that
			// we have no idea about this IR
		return
```

##### NIC/ASIC

The NIC/ASIC module consists of two separate processes, `NIC_ASIC_DOWNSTREAM` and `NIC_ASIC_UPSTREAM`, which receive/send packets from/to other switches or controller respectively.

- Downstream process (packet-in)
```pseudo
swNicAsicProcPacketIn: 
	OVER ({NIC_ASIC_IN} \X SW)
	VARIABLES "ingressIR" Received PACKET-IN message
	
	SwitchRcvPacket:
		WHILE TRUE
			DO
			AWAIT "messages exist in controller2Switch" AND "can receive packets"
			LET "ingressIR" BE "Head of controller2Switch"
			"Acquire switch lock"
			"Remove head of controller2Switch"
			
	SwitchNicAsicInsertToOfaBuff:
		IF "switch can recive packets"
			THEN "Pass lock to OFA_IN process" AND "Append ingressIR to NicAsic2OFA"
			ELSE "Return to SwitchRcvPacket"
```

- Upstream process
```pseudo
swNicAsicProcPacketOut:
	OVER ({NIC_ASIC_OUT} \X SW)
	VARIABLES "egressMsg" Output ACK
	
	SwitchFromOFAPacket:
		WHILE TRUE
			DO
				AWAIT "Can receive packets" AND "Message exist in Ofa2NicAsic"
				LET "egressMsg" BE "Head of Ofa2NicAsicBuff"
				"Acquire switch lock"
				"Remove head of Ofa2NicAsicBuff"
				
	SwitchNicAsicSendOutMsg:
		IF "switch can receive packets"
			THEN
				"Wait for lock"
				"Release lock"
				"Append new ACK to switch2Controller"
			ELSE "Return to SwitchFromOFAPacket"
```

##### OFA

Similar to NIC/ASIC, OFA also consists of two separate processes. These are:
- `OFA_DOWNSTREAM` extracts the IR, and sends it to the installer
- `OFA_UPSTREAM` waits for confirmation from Installer, and upon receiving it, sends a `INSTALLATION_CONFIRMATION` message to controller.

- `ofaModuleProcPacketIn` for OFA downstream
```pseudo
ofaModuleProcPacketIn
	OVER ({OFA_IN} \X SW)
	VARIABLES "ofaInMsg" Message from the NIC/ASIC
	
	SwitchOfaProcIn:
		WHILE TRUE:
			DO:
				AWAIT "OFA can process IRs" AND "Message exists in NicAsic2OfaBuff"
				"Acquire switch lock"
				LET "ofaInMsg" BE "Head of NicAsic2OfaBuff"
				"Pop NicAsic2OfaBuff head"
				
	SwitchOfaProcessPacket:
		IF "OFA can process IRs"
			THEN
				"Pass lock to installer"
				IF "ofaInMsg.type" IS "INSTALL_FLOW"
					THEN "Append "ofaInMsg.IR" to Ofa2InstallerBuff"
				ELSE
					EXPLODE (this shouldn't happen!)
			ELSE "Return to SwitchOfaProcIn"
```
- `ofaModuleProcPacketOut` for OFA upstream
```pseudo
ofaModuleProcPacketOut
	OVER ({OFA_OUT} \X SW)
	VARIABLES "ofaOutConfirmation" Ack message from OFA
	
	SwitchOfaProcOut:
		WHILE TRUE:
			DO:
				AWAIT "OFA can process IRs" AND "Message exists in Installer2OfaBuff"
				"Acquire switch lock"
				LET "ofaOutConfirmation" BE "Head of Installer2OfaBuff"
				"Pop Installer2OfaBuff head"
				
	SendInstallationConfirmation:
		IF "OFA can process IRs"
			THEN
				"Pass switch lock to NIC_ASIC_OUT"
				"Append INSTALLED_SUCCESSFULLY to Ofa2NicAsicBuff"
			ELSE "Return to SwitchOfaProcOut"
```
##### Installer

Installer has only one process unlike the above, which installs IRs in the data plane and returns a confirmation:

- `installerModuleProc`:
```pseudo
installerModuleProc
	OVER ({INSTALLER} \X SW)
	VARIABLES "installerInIR" The IR to install
	
	SwitchInstallerProc:
	WHILE TRUE:
		DO
			AWAIT "Switch can install IRs" AND "Message exists in Ofa2InstallerBuff"
			"Get switch lock"
			LET "installerInIR" BE "Head of Ofa2InstallerBuff"
			"Remove installerInIR from Ofa2InstallerBuff"
			
	SwitchInstallerInsert2TCAM:
		IF "Switch can install IR"
			THEN
				"Get switch lock"
				"Append installerInIR to installedIRs"
				"Put installerInIR in TCAM"
		ELSE "Return to SwitchInstallerProc"
		
	SwitchInstallerSendConfirmation:
		IF "Switch can install IRs"
			THEN
				"Pass lock to OFA_OUT"
				"Append installerInIR to Installer2OfaBuff"
			ELSE "Return to SwitchInstallerProc"
```
##### Failures

- `swFailureProc`:
  The process will allow each switch to fail independently. The process just waits until it is enabled and then proceeds to branch out all behaviors for any possible element failure. 
  This means that we can specify failure of all elements at once, or one after the other. Depending on which element failed, the appropriate macro defined previously will be called, and then the process awaits for the resolving process to start.
```pseudo
swFailureProc
	OVER ({SW_FAILURE_PROC} \X SW)
	VARIABLES "notFailedSet" Set of switch elements still alive
			  "failedElem" Temporary element storage variable
	
	SwitchFailure:
		WHILE TRUE
			DO
				LET "notFailedSet" BE "returnSwitchElementsNotFailed"
				\* I. there is an element to fail
				\* II. the system has not finished installing all the IRs
				\* III. either lock is for its switch or no one has the lock
				\* IV. there is no difference if switch fails after the 
				\*     corresponding IR is in IR_DONE mode
				\* V. switches fail according to the order of 
				\*    sw_fail_ordering_var (input), so this switch should be 
				\*    at the head of failure ordering sequence.
				
				AWAIT 
					"notFailedSet is not empty" AND 
					"Not yet finished" AND
					"Controller released the lock" AND
					EITHER "This switch has the lock" OR "No switch has the lock" AND
					"There exists and IR for the switch, which is not done" AND
					"This switch is the head of sw_fail_ordering_var"
				BRANCH WITH
					LET "failedElem" BE IN "notFailedSet" 
					
				IF "failedElem" IS "cpu"
					THEN
						AWAIT PROCESS "SwitchResolveFailure";
						cpuFailure();
				ELSIF "failedElem" IS "ofa"
					THEN
						AWAIT PROCESS "SwitchResolveFailure";
						ofaFailure();
				ELSIF "failedElem" IS "installer"
					THEN
						AWAIT PROCESS "SwitchResolveFailure";
						installerFailure();
				ELSIF "failedElem" IS "nicAsic"
					THEN
						AWAIT PROCESS "SwitchResolveFailure";
						nicAsicFailure();
				ELSE
					"Explode!, this shouldn't happen"
```
- `swResolveFailure`:
  This process resolves a failure to keep the specification going forward.
```pseudo
swResolveFailure
	OVER ({SW_RESOLVE_PROC} \X SW)
	VARIABLES "failedSet" Set of failed elements
			  ""
```

### OFC

#### Definitions

The OpenFlow controller has internal submodules:
- **Worker Pool:** A pool of worker processes that send the instructions (think of them as Netty threads in Java)
- **Event Handler:** Triggers the procedure for all events, including but not limited to failures
- **Monitoring Server:** Monitors switch status and checks for keep-alive messages from the switch to monitor failure conditions
- **Watchdog:** (Not unique to just the OFC) A module that restarts the internal modules when they go down

>[!IMPORTANT] Constrain On Failures
>Failure of each submodule/module may happen independently, **with the exception of the watchdog submodules**, as not being able to restart failed modules, leads to vacuous results, where the control plane just instantly shuts down and does nothing.

>[!FAQ] Locks
>To control when and how actions are enabled in a multi-module specification, internal submodules share a lock, and only when that lock is in the possession of the submodule, can it run certain actions.
>Locks also allow for the synchronization of actions between multiple threads in a submodule. Actions between separate modules however are asynchronous by definition.
>Note however that locks are not here to guarantee safety like their real world counterparts, here the increase efficiency by reducing the state space. 
>See [[#Macros And Helper Definition]] for details.

Similar to the switch, the OFC also has internal processes defined the same way. So we have:
```
ContProcSet = ({rc0} \X {CONT_SEQ}) \cup
			  ({ofc0} \X {CONTROLLER_THREAD_POOL}) \cup
			  ({ofc0} \X {CONT_EVENT}) \cup
			  ({ofc0} \X {CONT_MONITOR})
```
To get a member of this set, you pass a sequence into the set identifier, like for example `ContProcSet[<<ofc0, th>>]`.

Here:
- `CONT_SEQ` is the set of sequencer threads
- `CONTROLLER_THREAD_POOL` is the set of IR installation threads
- `CONT_EVENT` is the set of event handler threads
- `CONT_MONITOR` is the set of monitoring threads

Think of all of these sets as just a set of model values.

The connection between the OFC and the switches is one of the primary functions of the control plane. Each message is first put on a queue, and then a worker thread listening for that message will pick it up at some point in the future and react to it.

We do however make things a bit easier in case of keep-alive messages. Instead of periodically exchanging them, we only send them when *the state changes*, otherwise we don't do anything about them.

So we have:
```
\* Queue for switch to OFC monitoring server
swSeqChangedStatus = <<>>,
\* Queue for OFC workers to switches
controller2Switch = [x \in SW |-> <<>>],
\* Queue for switches to OFC
switch2Controller = <<>>
```
Multiple threads may attempt to work on an IR. We don't want to let IRs be reinstalled as much as possible to reduce the amount of possible behaviors. To this end, we will create a lock for individual IRs and keep them in a list:
```
idThreadWorkingOnIR = [x \in 1..MaxNumIRs |-> IR_UNLOCK],
```
We also create an auxiliary variable that assigns an arbitrary precedence to threads, that let's us choose between threads that want to lock on the same IR at the same time. This also helps reduce the number of behaviors.
```
workerThreadRanking = 
	CHOOSE x \in [CONTROLLER_THREAD_POOL -> 1..Cardinality(CONTROLLER_THREAD_POOL)]: 
		~\E y, z \in DOMAIN x: y # z /\ x[y] = x[z],
```
Here, `CONTROLLER_THREAD_POOL` is a set of model values that signify the number of threads available to the OFC.

##### Workers

We define the following for simplicity:
```
isSwitchSuspended(sw) == SwSuspensionStatus[sw] = SW_SUSPEND
```
The OFC workers will attempt to grab an IR, assign it to themselves and send it to the switch. This is where the OpenFlow implementation of our controller and the Southbound API will actually come in play when we implement this.

For now, the following definitions are needed for the actual process:
```
setFreeThreads(CID) == 
	{y \in CONTROLLER_THREAD_POOL: 
		/\ NoEntryTaggedWith(<<CID, y>>)
		/\ controllerSubmoduleFailStat[<<CID, y>>] = NotFailed}
```
A thread may only attempt to grab a scheduled IR, if and only if it has no IR scheduled for it at the moment. The above definition gives us the set of threads in `CONT_SEQ` that can be assigned to handle an IR. Of course, these threads need to:
- Be active at the moment, therefore `controllerSubmoduleFailState[<<CID, y>>] = NotFailed`. Remember that the CID was thread type identifier (i.e. `ofc0`, or `rc0`; here, we consider `rc0`).
- Have no entries in the NIB for them

Multiple threads may attempt to get the same untagged entry in the NIB. To break tie in this case, the thread with the lowest ID value is prioritized over the rest. Remember that thread identifiers were of the form `<<CID, modelID>>`, where `CID = {ofc0, rc0}` and `modelID` was a model value in the set of threads defined in the beginning (i.e. `CONTROLLER_THREAD_POOL`, `CONT_SEQ`, etc.).

The following definition will handle this assignment:
```
canWorkerThreadContinue(CID, threadID) == 
	\/ \E x \in rangeSeq(IRQueueNIB): x.tag = threadID
	\/ /\ \E x \in rangeSeq(IRQueueNIB): x.tag = NO_TAG
	   /\ NoEntryTaggedWith(threadID)
	   /\ workerThreadRanking[threadID[2]] = 
		   min({workerThreadRanking[z]: z \in setFreeThreads(CID)})
```
Continuing the above, we also define:
```
setThreadsAttemptingForLock(CID, nIR, IRQueue) == 
	{x \in CONTROLLER_THREAD_POOL: 
		/\ \E y \in rangeSeq(IRQueue): /\ y.IR = nIR
									   /\ y.tag = <<CID, x>>
		/\ pc[<<CID, x>>] = "ControllerThread"}

threadWithLowerIDGetsTheLock(CID, threadID, nIR, IRQueue) ==
	workerThreadRanking[threadID[2]] = 
		min({workerThreadRanking[z]: 
			z \in setThreadsAttemptingForLock(CID, nIR, IRQueue)}
			)
```
Here, `nIR` is the number of the corresponding IR. 
The above definitions may seem redundant, but remember that threads may push entries into the NIB requesting for the same IR number. The first definition gets the set of these threads, by finding ones that:
- Have active entries in the NIB for the same IR number
- Are tagged for the same thread
- Belong to the OFC process

The second definition resolves tie between multiple threads by choosing the one with the lowest numerical ID.

##### Event Handler

Multiple event handler instances appear within different modules, we'll define the processes later, but in general, the following apply to all modules.

Remember that `swSeqChangedStatus` is the buffer for messages from switches to OFC monitoring server, and this buffer will contain events of different types, and different statuses for that particular type.

Event types are:

| Type Handler              | Type Semantic                                              |
| ------------------------- | ---------------------------------------------------------- |
| `INSTALL_FLOW`            | From OFC to SW, carrying IR to install                     |
| `RECEIVED_SUCCESSFULLY`   | SW ACK for `INSTALL_FLOW`                                  |
| `INSTALLED_SUCCESSFULLY`  | SW ACK for flow installation in the data plane             |
| `KEEP_ALIVE`              | TCP Keep-alive                                             |
| `RECONCILIATION_REQUEST`  | Request for state reconciliation from controller to switch |
| `RECONCILIATION_RESPONSE` | Response from SW to `RECONCILIATION_REQUEST`               |
| `STATUS_NONE`             | Place holder type for unfinished request                                                           |
| `NIC_ASIC_DOWN`  | TCP connection with the controller is lost                  |
| `OFA_DOWN`       | SW reports that OFA is down. No flow may be processed now.  |

Even statuses are concerned with events about switch internal modules. These include:

| Status Handler   | Status Semantic                                             |
| ---------------- | ----------------------------------------------------------- |
| `INSTALLER_DOWN` | SW reports that flows cannot be installed in the data plane |
| `INSTALLER_UP`   | SW reports that installer module has recovered/is OK                                                            |

All events are numbered, which can be thought of as just timestamps. Multiple events for a single switch may be queued up at once, which necessitates that we use the most *recent* event for some processes. This event number will be useful here, which we designate with `num`.

Now, the definitions are as follows:
```
existsMonitoringEventHigherNum(monEvent) == 
	\E x \in DOMAIN swSeqChangedStatus: 
		/\ swSeqChangedStatus[x].swID = monEvent.swID
		/\ swSeqChangedStatus[x].num > monEvent.num

shouldSuspendSw(monEvent) == 
	\/ monEvent.type = OFA_DOWN
	\/ monEvent.type = NIC_ASIC_DOWN
	\/ /\ monEvent.type = KEEP_ALIVE
	   /\ monEvent.status.installerStatus = INSTALLER_DOWN

canfreeSuspendedSw(monEvent) == /\ monEvent.type = KEEP_ALIVE
								/\ monEvent.status.installerStatus = INSTALLER_UP

getIRSetToReset(SID) == 
	{x \in 1..MaxNumIRs: /\ IR2SW[x] = SID
						 /\ IRStatus[x] \notin {IR_DONE, IR_NONE}}
```

The first 3 definitions are intuitive. The last definition will be used for NIB cleanup or reconciliation. We'll see it's usage later.

##### Failure Recovery

Similar to switches, here we also define:
```
controllerSubmoduleFailNum = [x \in {ofc0, rc0} |-> 0],
controllerSubmoduleFailStat = [x \in ContProcSet |-> NotFailed],
```
Here, failure can always be detected with a watchdog module in the controller.
The following auxiliary macros also help:
```
moduleIsUp(threadID) == controllerSubmoduleFailStat[threadID] = NotFailed
controllerIsMaster(controllerID) == 
	CASE controllerID = rc0 -> masterState.rc0 = "primary"
	  [] controllerID = ofc0 -> masterState.ofc0 = "primary"

getMaxNumSubModuleFailure(controllerID) == 
	CASE controllerID = rc0 -> MAX_NUM_CONTROLLER_SUB_FAILURE.rc0
	  [] controllerID = ofc0 -> MAX_NUM_CONTROLLER_SUB_FAILURE.ofc0
```

>[!TODO]
>The variable `masterState` has no logic attached to it. 
>There is some in lines 723 - 734, but it's been commented out. Ask Pooria at some point ...

The constant `MAX_NUM_CONTROLLER_SUB_FAILURE` is a function of RE and OFC threads, and maps them into a maximum number of failures.
It is defined at the beginning of the simulation:
```
CONSTANT MAX_NUM_CONTROLLER_SUB_FAILURE
```

#### Macros

- `controllerModuleFails()`
	- Atomically forces submodule failure in the controller.
- `controllerModuleFailOrNot()`
	- Called during each step of the controller.
	- Non-deterministically causes a submodule to fail by calling `controllerModuleFails` or does nothing.
	- Only effective when the module is in `NotFailed` and the maximum number of failures (i.e. `getMaxNumSubModuleFailure(self[1])`) has not been reached. 
- Skipped 765
- `controllerSendIR(s)`
	- Mimics the process of sending the IR to a switch.
	- Works by putting a `INSTALL_FLOW` message in the `controller2Switch` queue of the switch assigned to the given IR `s`.
	- Atomically releases `switchLock` and gives it to the `NIC_ASIC_IN` process of the corresponding switch.

#### Procedures

- Schedule IRs
```pseudo
scheduleIRs:
	INPUT: "setIRs" Set of IRs to schedule
	VARIABLES: "nextIR"
	
	SchedulerMechanism:
		WHILE "setIRs isn't empty"
			DO "Set nextIR to be some IR in setIRs"
			ASSERT "nextIR status is either IR_NONE or IR_DONE"

			AddToScheduleIRSet:
				ASSERT "nextIR isn't in SetScheduledIRs";
				"Add nextIR to SetScheduledIRs"
				// Use IR2SW to find the IR in SetScheduledIRs
		
			ScheduleTheIR:
				"Add nextIR to set of scheduled IRs for this thread"
				"Remove nextIR from setIRs"
	return
```

### RE

#### Definitions

The Routing Engine (or RC, *routing controller*), accepts intents from higher applications, converts them to DAGs and then sends them to the sequencer for installation. Instead of specifying DAGs themselves however, we do something much more general.

We define an upper bound on the number of IRs as `MaxNumIRs`. Now, we generate all possible non-isomorphic DAGs with at most `MaxNumIRs` nodes and use them to generate all possible behaviors up to some acceptable point.

To do this, a definition `generatedConnectedDAG(S)` is used:
```
generateConnectedDAG(S) == 
	{x \in SUBSET (S \X S): /\ ~\E y \in S: 
								<<y, y>> \in x
							/\ ~\E y, z \in S: 
								/\ <<y, z>> \in x
								/\ <<z, y>> \in x
							/\ \A y \in S: 
								~\E z \in S: 
									/\ <<z, y>> \in x
									/\ z >= y
							/\ \/ Cardinality(S) = 1
							   \/ \A y \in S: 
								   \E z \in S: 
									   \/ <<y, z>> \in x
									   \/ <<z, y>> \in x
							/\ \/ x = {}
							   \/ ~\E p1, p2 \in Paths(Cardinality(x), x): 
								   /\ p1 # p2
								   /\ p1[1] = p2[1]
								   /\ p2[Len(p2)] = p1[Len(p1)]
							/\ Cardinality(x) >= Cardinality(S) - 1
							/\ \/ x = {}
							   \/ \A p \in Paths(Cardinality(x), x): 
								   \/ Len(p) = 1
								   \/ p[1] # p[Len(p)]}
```
Here, `Paths(n, S)` generates the set of paths from nodes in `S` with length `n`. The DAG will dictate the ordering of IRs, after this, we need only generate a mapping from switches to IRs. So in brief, we can do:
```
switchOrdering = 
	CHOOSE x \in [SW -> 1..Cardinality(SW)]: ~\E y, z \in SW: y # z /\ x[y] = x[z],
dependencyGraph \in generateConnectedDAG(1..MaxNumIRs),
IR2SW = 
	CHOOSE x \in [1..MaxNumIRs -> SW]: ~\E y, z \in DOMAIN x:
		/\ y > z
		/\ switchingOrder[x[y]] =< switchOrdering[y[x]]
```

##### Sequencer 

The sequencer will schedule IRs according to the DAG. Therefore, we define:
```
isDependencySatisfied(ir) == 
	~\E y \in 1..MaxNumIRs: /\ <<y, ir>> \in dependencyGraph
							/\ IRStatus[y] # IR_DONE

getSetIRsCanBeScheduledNext(CID) == 
	{x \in 1..MaxNumIRs: /\ IRStatus[x] = IR_NONE
						 /\ isDependencySatisfied(x)
						 /\ SwSuspensionStatus[IR2SW[x]] = SW_RUN
						 /\ x \notin SetScheduledIRs[IR2SW[x]]}
```
The first one, checks whether or not all dependencies of a given IR have been satisfied. The second one returns the set of possible IRs that may be scheduled. Note that any IR in this set must have:
- Never been scheduled before
- It's corresponding switch must be running
- Must have satisfied dependencies
- Must be in `IR_NONE` state



### NIB

#### Definitions

The NIB communicates with both the OFC and the RE, thus we need queues here as well. A queue for all scheduled IRs being process is needed which we denote as `IRQueueNIB`. 
We'll also need one for the state of each IR for reconciliation (`SwSuspensionStatus`), for process state (`controllerStateNIB`) and of course, one for the set of scheduled IRs (`SetScheduledIRs`) within a switch to inform the RE that the IR is done.
```
IRQueueNIB = <<>>,
controllerStateNIB = [x \in ContProcSet |-> [type |-> NO_STATUS]],
IRStatus = [x \in 1..MaxNumIRs |-> IR_NONE],
SwSuspensionStatus = [x \in SW |-> SW_RUN],
SetScheduledIRs = [y \in SW |-> {}],
```
These are the main NIB variables that we need for this spec. The queue in particular will list who is supposed to handle what IR that is supposed to be scheduled. To this end, all unscheduled entries in the NIB will be tagged with `NO_TAG` at first.
```
CONSTANT NO_TAG
```
The buffer will contain tagged entries of the scheduled IRs. Each entry in the queue will have the following attributes:
- `tag`: The thread assigned to the entry
- `IR`: The IR associated with this entry

Note that threads are processes as well, which can always be identified with `self` locally.

##### NIB Operations

Some helper definitions for NIB are as follows:
```
NoEntryTaggedWith(threadID) == ~\E x \in rangeSeq(IRQueueNIB): x.tag = threadID

FirstUntaggedEntry(threadID, num) == 
	~\E x \in DOMAIN IRQueueNIB: /\ IRQueueNIB[x].tag = NO_TAG
								 /\ x < num
```
We'll use the second one to query whether the given index is the first untagged entry in the queue or not. This will be useful for traversing the contents of the buffer.

The above two will be used to define the following:
```
getFirstIRIndexToRead(threadID) == 
	CHOOSE x \in DOMAIN IRQueueNIB: \/ IRQueueNIB[x].tag = threadID
									\/ /\ NoEntryTaggedWith(threadID)
									   /\ FirstUntaggedEntry(threadID, x)
									   /\ IRQueueNIB[x].tag = NO_TAG

getFirstIndexWith(RID, threadID) == 
	CHOOSE x \in DOMAIN IRQueueNIB: /\ IRQueueNIB[x].tag = threadID
									/\ IRQueueNIB[x].IR = RID
```

You can think of the NIB as a table. Rows are thread identifiers (i.e. tags), and the columns will contain the IR associated with it. The basic read/delete/insert operations of the NIB can now be defined like the following:
```
macro modifiedEnqueue()
begin
	IRQueueNIB := Append(IRQueueNIB, [IR |-> nextIR, tag |-> NO_TAG]);
end macro;

macro modifiedRead()
begin
	rowIndex := getFirstIRIndexToRead(self);
	nextIRToSent := IRQueueNIB[rowIndex].IR;
	IRQueueNIB[rowIndex].tag := self;
end macro;

macro modifiedRemove()
begin
	rowRemove := getFirstIndexWith(nextIRToSent, self);
	IRQueueNIB := removeFromSeq(IRQueueNIB, rowRemove);
end macro;
```
The definition of `removeFromSeq` is:
```
removeFromSeq(inSeq, RID) == 
	[
		j \in 1..(Len(inSeq) - 1) |-> IF j < RID 
									   THEN inSeq[j] 
									   ELSE inSeq[j+1]
	]
```
Where `RID` is a row ID, this just removes an index from the sequence.

Remember that these macros are defined within the NIB processes. Therefore variables like `nextIR` will be defined locally once we reach them. For now, just know:

- `Enqueue` will insert a new entry into the queue, that is untagged and scheduled for the current IR under operation.
- `Remove` will delete the first index in the queue belonging to the calling thread.
- `Read` will get the first item in the queue that is unscheduled (i.e. not tagged) or the first item in the queue that belongs to the calling thread, and assigns it to the calling thread.

All of these functions are atomic operations per [[PlusCal]] definitions.



### Macros And Helper Definitions

#### Definitions
```
whichSwitchModel(swID) == WHICH_SWITCH_MODEL[swID]

swCanReceivePackets(sw) == switchStatus[sw].nicAsic = NotFailed

swOFACanProcessIRs(sw) == /\ switchStatus[sw].cpu = NotFailed
						  /\ switchStatus[sw].ofa = NotFailed

swCanInstallIRs(sw) == /\ switchStatus[sw].installer = NotFailed
					   /\ switchStatus[sw].cpu = NotFailed

getSetIRsForSwitch(SID) == {x \in 1..MaxNumIRs: IR2SW[x] = SID}
```
All above definitions are intuitive. Besides theses, we also have the definition used for watchdog modules:
```
returnControllerFailedModules(cont) == 
	{x \in ContProcSet: /\ x[1] = cont
						/\ controllerSubmoduleFailStat[x] = Failed}
```
Remember that for each element in `ContProcSet` (which are tuples), the first element is either `rc0` for RE threads, or `ofc0` for OFC threads.

And the definition for a termination condition:
```
isFinished == \A x \in 1..MaxNumIRs: IRStatus[x] = IR_DONE
```

The following macros are also relevant to the implementation of many processes. We don't discuss the code here for most of these, we just say where they are and what they do:

- `removeFromSeqSet(SeqSet, obj)` 
	- Defined in *444*
	- Remove object `obj` from a sequence of sets `SeqSet`

#### Macros

Some general macros are used to implement synchronization between processes. As we mentioned, the implementation uses a lock, but this lock serves no safety purposes, it only removes unnecessary interleaving between processes, and thus speeds up the verification process (in contrast to it's function in the real world, which actually slows down the process).

Locks are *given* to a process, when it's identifier is assigned to that particular lock. In the particular case that a process does not have a lock, a constant identifier `NO_LOCK` will serve as a placeholder. Thus, locks are either a process name (i.e. a tuple), or the sequence `<<NO_LOCK, NO_LOCK>>`.

Locks exist for the controller processes (i.e. `contorllerLock`) and switches (i.e. `switchLock`).

The following macros can be defined then:

- `switchWaitForLock()`
	- Awaits for `controllerLock` to be released
	- Awaits for `switchLock` to be equal to `self` or be released
- `acquireLock()`
	- Waits for lock with `switchWaitForLock()` and then assigns `self` to `switchLock`. 
	- This is only used for switches
- `acquireAndChangeLock(nextLockHolder)`
	- Waits for lock with `switchWaitForLock()`
	- Passes the lock to `nextLockHolder` by assigning it to `switchLock`
- `releaseLock(previousLockHolder)`
	- Enabled only when either the lock is free, or is actually really owned by `previousLockHolder`
	- Atomically releases the lock from another process
- `controllerWaitForLockFree()`
	- Awaits for when both `controllerLock` and `switchLock` are FREE.
	- Basically, the controller only proceeds when the switches and it's internal processes are done with their current tasks (this includes causing failures as well!)
- `controllerReleaseLock(prevLockHolder)`
	- Similar to `releaseLock(previousLockHolder)`, but releases the lock for controller processes instead.

## Invariants

### Consistency

All IRs will eventually be in the `IR_DONE` state. Thus to simulate this, a variable will record the set of all IRs installed on a switch, so `installedIRs = <<>>`.