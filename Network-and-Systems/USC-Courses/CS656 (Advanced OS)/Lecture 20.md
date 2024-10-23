# Code Offload

**Paper:** [MAUI: Making Smartphones Last Longer with Code Offload](https://dl.acm.org/doi/pdf/10.1145/1814433.1814441)

This paper is kind of a continuation of the previous paper with a different view, but using a similar idea. Offloading power hungry tasks to a remote executor.
We saw this in the previous paper in the context of the speech recognition task, where the bigger model was executed remotely, whereas only a small model at most would be executed locally which would only do pre-processing.

So, can we generalize this approach? Specifically for smartphones?
## Naïve Approaches

- **Run apps inside a VM**, migrate VM between phone and cloud when needed
	- Too much overhead to migrate whole application stacks back and forth
	- When to offload is a question that is really difficult ..
- **Programmer partitions app**, some parts run on the phone, some parts in the cloud.
	- Puts a lot on the plate of the programmers
	- Optimal partitioning might have to be done dynamically, but most approaches are static

So here is the main idea, offload at *function level*, not a whole code or module and dynamically identify when to offload.

![[Pasted image 20240327141703.png|500]]

## Key Questions

There are at least 3 main things to address:
1. Which functions to offload?
2. When to offload?
3. How to offload?

### What To Offload?

Programmer should specify which function can be offloaded. The main reason that the programmer needs to get involved with is that some functions should NOT be offloaded:
- GUI calls
- Local hardware calls
- Functions that cannot be re-executed if offload fails

The other side of this approach, letting programmers specify functions that should remain *local* rather than remote can be incorrect. If the programmer misses some function that MUST run locally, then offload can break program correctness.
Usually, if a function can be offloaded, it should be able to run locally. If it runs locally, it may not necessarily run as an offload.

### When To Offload?

At the very least, you need 3 things:
1. Client characteristics (maybe with an offline benchmark)
2. Program behavior (requires code performance benchmarks)
3. Network characteristics (RTT is really important)

![[Pasted image 20240327143215.png|500]]

A solver needs to find out the benefit of offloading. Some model will be needed to estimate the  energy consumption of sending data over a specific bandwidth with a specific size. Given this data, we can realize how much energy can be saved if we knew how many CPU cycles are needed to execute a function remotely or locally.

But there are 2 big problems:
- Identifying the size of the state to transfer must be done dynamically and so it takes time.
- CPU energy model depends on runtime, how to estimate this runtime without running the function?

The solution is to just rely on the history of executions.

>[!IMPORTANT]
>Sequential offload of functions may not be worth it. It is important to at times, group functions and offload them as a pipeline.

### How To Offload?

Essentially, this requires two version of an application. One runs on a phone and the other run in the cloud.

Here however, we come to the main reason that function offload is more efficient than a whole VM, and that is because we only need to transfer *changes* in function state since the previous offload, rather than all of it.

![[Pasted image 20240327150101.png|500]]

## Evaluation

- The authors use both a micro/macro benchmark for their tests.
- 4 types of applications were tested:
	- Face recognition
	- Chess
	- Highly interactive video game
	- Real-time voice translator from Spanish to English
- Metrics profiles:
	- Energy consumption
	- Execution duration
	- Offloaded data

### Macrobenchmark

![[Pasted image 20240327152123.png|700]]

Above, we can see the energy consumption for 3 of our applications except the voice translators. Here, we have 3 types of runs:
- Local run (First bar)
- Four different MAUI runs with increasing network RTT (More RTT means more runtime and so more energy)
- The final one is a special version of MAUI that runs on 3G with 220 milliseconds of RTT

![[Pasted image 20240327152520.png|700]]

We can also measure the execution time like above:
- There is drastic saving for the face recognition
- Latency is heavily decreased for the video game task
- For higher RTTs, the chess task is not worth it

### Microbenchmarks

We investigate:
- The solver overhead
- Effect of incremental deltas for sending function state
- How MAUI copes with network condition changes

The MAUI solver is just an Integer LP that can be solved very quickly, in a few tens of milli-seconds. The solver is a very important part of the program, since if it misbehaves or takes too long, code may not be offloaded despite nothing preventing it.

![[Pasted image 20240327153529.png|400]]

For evaluating state delta effectiveness, we can also just disable it and use the state as a whole:

![[Pasted image 20240327153613.png|500]]

For a network/CPU microbenchmark, the authors turned to the video game tasks. The tasks in the game are:
- `DoLevel()`: Load the map and assets
- `HandleEnemies()`: Handle enemy AI
- `HandleMissiles()`: Handle moving objects towards the player
- `DoBonuses()`: Reward the player

Two scenarios were used:
- Scenario 1: Less computation, no missiles, just load the intro and the level
	- MAUI tolerates increased RTT
- Scenario 2: More computation, spawn more missiles
	- MAUI will not tolerate large RTTs and will not offload the `DoLevel` task if the RTT is too large