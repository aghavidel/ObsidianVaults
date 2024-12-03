**Paper:** [Ray: A Distributed Framework for Emerging AI Applications](https://www.usenix.org/system/files/osdi18-moritz.pdf)

# Ray

We are shifting the focus compared to previous lectures. We are now focusing on distributed programming models. We've already seen Spark, which is coincidentally created by the same group at Berkley that made the system de jour (i.e. RISELab, supervised by Ion Stoica).

## Background

The main thing that concerns the system that we want to design, is very large scale training of Reinforcement Learning (RL) models.
RL systems are unique:

- The data that they train on is computationally expensive, since it comes out of simulations.
- Having multiple simulations done in parallel is extremely important for training models.

![[Pasted image 20241107121248.png|500]]

The general workflow is:
- Some data structure (our model), has a *policy* embedded into it that allows it to make decisions based on observations
- Once an *action* is made, it is emitted to the *environment* (i.e. a simulation) and observations are made. **An observation is an environment state and a reward value**.
- A list of observations (which we call a *trajectory*) results from repeating the above in a (perhaps parallel) loop. This trajectory is our training data.
- Some training algorithm is used to consume the trajectory and update the policy.

Any infrastructure that wants to handle this workload needs to have some properties:
- Low latency for task submission and high total throughput
- Support dependencies on tasks, and complex logic based on how tasks progress
- Fault tolerance when it comes to tasks failing or machines crashing
- Seamless support for a heterogenous environment of GPUs and other accelerators

Some RL definitions:
- An *environment* is some system with an internal configuration that we want to communicate with and manipulate. This is what we simulate.
- A *state* is a data structure that fully defines the current configuration of the environment
- A *policy* is some model that accepts a *state* and outputs an *action*, which can update the current configuration of the environment.
- A *reward*, is some value that is calculated for each action that is applied on an environment. The details of how a reward is calculated can be arbitrarily complex.
- `rollout` is the procedure of applying an action and getting back an *observation*, a tuple of state and reward values.
- A *trajectory* is a list of consecutive observations.
- `train` is the procedure of consuming a trajectory and updating the policy ... *somehow* ...

## Ray Architeture

Now, some *ray* definitions:
- **Tasks**: These are (perhaps remote!) function calls. They accept a series of arguments and output a **read only value** as a result.
  As such, they are very good for immediate, stateless result generation, but not good for updating things repeatedly.
  They are idempotent by definition, and as such, trivially fault tolerant.
- **Actors**: These are stateful classes, they can have arbitrary methods to manipulate their internal state. They are much harder to make fault tolerant, as they need checkpointing.

![[Pasted image 20241107123954.png|500]]

Here is an example:

```python
# A task
@ray.remote
def increment(a):
	return a + 1

# An actor
@ray.actor
class Incrementer:
	def __init__(self):
		self.value = 0
	def inc(self):
		self.value += 1
		return self.value

"""
This is how to use tasks
"""
res1 = increment(0)
res2 = increment(res1)
res3 = increment(res2)

"""
This is how actors are used
"""
actor = Incrementor()
res1 = actor.inc()
res2 = actor.inc()
res2 = actor.inc()
```

In the above, `resi` objects are actually *handles*, they refer to some object in remote memory.
In Ray, these are kept in a Redis datastore, a node called *Global Control Store* (GCS). The GCS maintains multiple tables:

- **Table of Objects:** Containing references and handles for all generated objects
- **Table of Actors and Tasks**: Containing references to all tasks and actors submitted

The data is kept read-only as much as possible. We cannot accept the burden of updating in place and synchronizing.

>[!FAQ] The Class Quiz Answers
>For global scehduling:
>- Progress of tasks is not used.
>- Location and Size of inputs is used and is queried from the GCS.
>- Serialized input data is not used. Just knowing the input size is enough.
>- The queue size and resource availability of a node is signaled from its heartbeat messages.

The GCS provides the shared memory needed to glue things together, but the question of where to actually do the computation still remains.

Computations are done in separate nodes that connection to GCS. Each node has its own local scheduler that tries to utilize its resources locally as much as possible, and only bother other nodes if it cannot feasibly handle the task by itself. To do this, a global scheduler exists that handles task and actor assignments to other nodes, thus, *scheduling has a hierarchy*, and this is the key for scheduling huge amounts of tasks per second.

![[Pasted image 20241107130827.png|500]]


## Ray Operation

Here is how a task is executed:

![[Pasted image 20241107131104.png|500]]

- Once `ray.remote` has annotated a function, a function object is created in GCS, which can then be assigned to some worker for future execution (in the above, it is assigned to node `N2`)
- When `add.remote`, i.e. the function above is actually called, it goes to the local scheduler and sees if it has a local worker to execute it. 
- No worker can locally handle the task, thus we bother the global scheduler, which queries the function table and realizes that a worker in `N2` can handle the task.
- The task then gets execute, which requires two arguments, `a` and `b`.  In the above, both `a` and `b` are remote objects (they have a reference in the object table in GCS). The object *reference* is in the GCS (i.e. it knows where each object is), but the object itself can be in remote nodes. In particular, it is in the *Object Store* of each node.
- In the above example, `b` is already in node 2, but `a` is in remote node 1. After querying the GCS, all objects required to execute the task can be made locally available in node `N2`, and once that is done, the task is execute and the result handle is written to GCS.

So, how to actually *get* the result?

![[Pasted image 20241107131923.png|500]]

Once `ray.get` is executed: 
- Again, we check with the local scheduler if we have the CPU resource to handle this. Before any of this though, we check the  local object store to see if the handle is actually already there (it isn't in the above!). This is done by querying the GCS to see where objects actually are.
- The result of the computation of the task, referred with $id_c$ in the above, is originally in the object store of `N2` (where it actually got generated in the first place).
- An RPC is executed that transfers the remote object of `c` in node `N2` to the local object store of `N1`. 
- Once the RPC is done, we get the result locally in node `N1` by just reading the value in object store.
