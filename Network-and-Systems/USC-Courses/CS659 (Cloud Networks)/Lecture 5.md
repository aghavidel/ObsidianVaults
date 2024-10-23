# A Throughput Centric View of Networking, Pooria Namyar

Some things about Data Centers:
- They live for a very long time, and as such, the traffic they receive changes heavily.
	- Thus, you might find it beneficial to create a DC that can handle any traffic
	- It is also good for users, since they can place their VM anywhere (kind of similar to unifmr bandwidth)

As such, we like **non-blocking topologies**, they provide this property.

## A Word on Bisection-Bandwidth

It is defined as the minumum number of links between two equal cuts of the network. Ideally, if the links have the exact same capacity, the condition for ideal utility is:

$$
BW \ge \frac{N}{2}
$$
Where $N$ is the number of servers. Any topology that satisifes this proprty is said to have **Full Bisection Bandwidth**. One popular realization is **Clos**. However, Clos is very regular and expensive, expanding it is very troublesome.

Thus, people looked into the other end of the spectrum.
- **Jellyfish**, no regularity, completely random
- **Expanders**, the core is regular but the rest is not

Expanders are very popular, they:
- Cost less
- Can whitsatand some failure
- Expand easily (duh!)

However, beyond Bisection Bandwidth (BB), there is also **Throuhput**, which is the analytic capacity of a topology with respect to a traffic matrix. It is only natural that we look to the **Throughput Upper Bound** with respect to all traffic matrices. 

The problem? **Throughput is extremely hard to calculate, and not at all feasible to find at a large scale**, it requires solving an optimization problem that is really expensive.

>[!FAQ] The question here is:
>Is B..B comparable to Throughput?
>The paper result is that **no!** A full BB network may not have a high throughput.
## A Word On Expanders

"INSER FIGURE FROM LECTURE"

In Clos, some switches do not connect to any switches (i.e. core switches), expanders however have the same number of servers for each switch (there is no real core).

To reason about these toplogies, we need to consider 2 types of traffics:
1. Trnsit traffic: Traffic that originates from one pod and ends up in anoter one (in Clos, core switches hanle exclusivly transit traffics)
2. Internal traffic wich just resolves in the pod itself

 "You know what, just read the paper!"

# VM Management in Clouds, Weiwu Pang

For the purposes of this discussion, a cloud is just a coputation/networing resource that is available on-demand. These are large sets of machines that are hosted by providers like Google and Micorsoft. We have already seen them to some extedn in the context of the Jupiter paper.

Clouds srvices differ significantly, but in a nutshell, people usually group them based on service models:
- There is Software-as-a-Service (SaaS), which is by all acounts the more familiar one. It is a tool or product that is available on-demand and persists some state after use. This would be your gmail, your drive, and Slack. Interactin with cloud resources is very limited.
- There is Platform-as-a-Service (PaaS), which allows some interaction with the cloud resources is somewhat available. This would be you MapReduce, your DFSs and container deployers (e.g. Kubernetes)
- There is Infrastructure-as-a-Service, like VPSs and VMs. These proved the highest degree of access to the cloud resources.

## Clouds From The POV of Clients

There are 2 things to consider, internal cloud connection and external connection:

- Clients usually specify the VMs for their projects.
- There can be many VMs (thousands!), and they would need private connection among themselves. Usually, the connection is provided with a Virtual Private Cloud (VPC). The VPC is a virtualized version of Googles network and cn provide connection among VMs that are event in different regions (i.e. East-West coast!)
- In the olden days, a project can only have one VPC, but now there can be multiple VPCs for one project if it is needed.
- ***Andromeda*** provides these VPC services.

To handle external connection of VMs:
- Connections that are on-premise (do not cross between regions) can be handled with a private VPN channel.
- There is also multi-cloud connections, that cross between different clouds and that can be handled by the provider with a dedicated network.
- Then there is site-to-site transfer, where a WAN s required, and for example, Google provides its own SD-WAN for this.
