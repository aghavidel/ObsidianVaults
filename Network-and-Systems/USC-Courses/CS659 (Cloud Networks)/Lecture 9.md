**Paper:** [Orion: Google's Software Defined Control Plane](https://www.usenix.org/system/files/nsdi21-ferguson.pdf)

# Intro.

Some background on network *control*:
- **Switch Control:** Dictate how traffic goes through the network by dictating flow entries
- **Network Control**: Dictate how network handles ingress traffic to satisfy certain policies

Orion is an evolution of the [[Onix]] controller that Google first used. Google introduced many new things into it and we are going to discuss them a bit.
Before that, we need to discuss the Microservice Architecture.

In brief, a microservice architecture would entail that each *function* of the system is dedicated to one single piece of software, ideally developed by a single team.
A microservice based system has some benefits:
1. It is much easier to scale, each function is independent and can be made beefier as needed
2. It allows independent development of the application
3. Can withstand failures much more easily

There are also problems:
- Possibly lower performance
- Much more complex, since we turn a monolithic application into a distributed one

![[Pasted image 20240207162327.png]]


Orion is Google's implementation of a microservice based controller. 

![[Pasted image 20240207163608.png]]

We have already seen the NIB in the context of the Onix controller, and it provides the same functionality here.

There are 4 main challenges with this design:
1. We need fast updates for NIB
	- Since NIB is gluing everything together, things can get out of hand very quickly
1. Lack of fate sharing among microservices
	- If two modules depend on each other, we need to make sure that they respond to the failure of the other module correctly
2. Limit failure radius
	- If some failure happens in the controller/data plane, we need to contain it as much as we can and make sure it does not cascade 
3. Integrate existing routing protocols into Orion
	- How to integrate a centralized design into a network that was previously purely distributed

## Intent-Based Control and Management

An **Intent** is a declarative statement that defines the end state epected for a system. For example:
- Connect points A and B over a 10 Gb link
- Drain router A

It only describes the goal, not how it is actually done.

In Orion, an operator (a human) will describe what the high level intent is, and the system will break it down to a series of intermediate intents by using the microservices.

## Aligning Failure Domains

- A failure domain is defined as a series of devices that can be expected to fail at the same time. An example of it would be any device connected to the same power cord.
- A control domain is defined as the set of the devices under the governance of the same controller.

A principal of the Orion design is that failure and control domains are kept essentially the same to keep the network operational even in the face of a catastrophic failure.

Beyond that, we also don't give a whole cluster to the same controller. We assign a separate controller to each spine block $S_i$ and each aggregation block $A_i$, and unify them with an intermediate block (the IBR). 

Graphically:

![[Pasted image 20240207171018.png]]

This limits blast radius (set of devices affected by a single failure).

## Optimistic Reaction To Switch Failure

On a non-SDN based system, the control and data plane share fate. If a switch fails, the control plane instance of it will also go down and we just have to route traffic around it.

On a system where the control plane and data plane are decoupled, things are more complicated.
Assume the following scenarios:
1. A switch failed
2. The out of band connection to the switch failed
3. Some component in the controller that handles switch connection failed

In all of these, the way we actually see the failure is that the switch will stop working, but the way that we should *react* to each failure is completely different. For the first one we should route away from the switch, but for the others we should let the switch be and resume the connection.

This allows us to define to general class of actions in the face of failure:
- **Fail Static (also Fail Open):** Keep switch functional (optimistic)
- **Fail Closed:** Route the traffic away from the switch (pessimistic)

Switches are given 3 states:
- **HEALTHY**: We have received messages from the switch and neighbors say that is OK
- **UNHEALTHY**: Either the switch or its neighbors say that things are not OK
- **UNKNOWN**: Anything else

In Orion, a high level policy for failure is:
- If we have many switches that are in UNKNOWN state, we fail static and not do anything 
- If we have a small number of switches in the UNKNOWN or UNHEALTHY state, we fail closed

>[!NOTE] Controller Network
>There are two choices for the controller network.
>1. In-Band: Use the data plane as the controller network (has circular dependencies since we need to setup routing first for the controller itself!)
>2. Out-Band: A separate network for the controller.
>
>Orion uses a hybrid approach.

