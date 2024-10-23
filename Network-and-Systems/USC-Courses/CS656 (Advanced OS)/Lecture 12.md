**Paper:** [Outatime](https://dl.acm.org/doi/pdf/10.1145/2742647.2742656)
# Speculative Execution

This paper is kind of a different scenario of execution in comparison with the previous paper.
The core obstacle for both of these is *Network Latency*, so how does it manifest?
- In **Sprite**, clients needed to access remote files for local execution 
- In this paper, **Outatime**, clients ask remote nodes to *do the execution for them*, and they only observe the results of the execution

Just to make sure we are on the same page, why do we need to do this? (in the context of *Outatime*, which is mobile gaming)
1. Clients just may not have the hardware available to play them (especially true on mobile devices)
2. It is much easier for the developers to assume that the game is only executed at the server, where they have full control over the hardware of the game (think consoles vs PC)
3. It is much easier (and safer!) to *update* software during runtime

Network latency is one of the biggest enemies of cloud gaming, since it breaks the loop on the client side and can drop inputs. One classic way of lowering the network latency between a client and a server is to *move the two closer together*.

Usually however, we can't move *clients* closer to servers, since:
- It could be very infeasible for the clients
- They just may decide its not worth the bother
- There are now laws and government regulations (especially in the EU) that can prevent this

The other way, while feasible, is still hard:
- Distributed set of servers are hard to manage
- At off-peak times, servers (especially nearby ones) tend to be underutilized
- Being close, does not necessarily mean lower latency! (just think of a busy LAN!)

So, how do we beat these odds?

## Intro.

One of the important numbers when it comes to gaming and live media streaming is always the FPS. Outatime assumes a minimum FPS requirement of at least 30, which implies that the user inputs must be accounted in at most 32 milli-seconds.

The problem is that the server RTT is variable and generally much larger than this amount.

![[Pasted image 20240221142103.png|500]]

Let's digest this picture:
- Time on the horizontal axis is divided to fixed size slots of 32 milli-seconds (depending on the FPS specification).
- A user inputs something on frame tick $t_5$
- It reaches the server on the tick of $t_7$, and the server (ideally!) would account for that input for the next tick, so the best time for getting the result if on $t_8$
- The result returns to the user on $t_{10}$

This interval between that first input on $t_5$, and the actual realization of it on the client side at $t_{10}$, which would be 5 ticks (i.e. 160 ms) is called (rather worryingly) **Frame Time**. Ideally, the Frame Time should be equal to the rendering tick which was 32 ms.

So how can we resolve this big difference?
Enter **Speculative Execution:**

![[Pasted image 20240221143023.png|500]]

The idea is:
- The server *predicts* the possible realizations for a whole RTT forward and send them to the user
- On the user side, if the prediction turns out to be correct, then the client immediately consumes the pre-computed input, shaving off a whole RTT worth of delay from the computation!
- If the prediction turns out to be incorrect, the server accepts the input and computes the result as expected.

Assuming perfect prediction of input and RTT, this guarantees immediate execution of all frames after the first RTT worth of execution.

## Prediction

The prediction of the user input is hard. The paper considers it over many different execution scenarios:

### Navigation

For navigation, we can just use a model (be that statistical or ML model) that is fed the history of the user input as is asked to output the possible realizations for an RTT into the future.

![[Pasted image 20240221143904.png]]


Navigation tends to be quite predictable. The parts that are not predictable are asynchronous and spontaneous actions that effect more than just movement. We'll see this in the next section. 

But this begs the question, what if we are wrong in our estimation?

On the server side, we know the full 3D space view of the client (think of a cube around a camera, with front-back, left-right and up-down faces). If we miss-predict , we can just rotate this mapping a bit and get the correct view. The problem however is that we can't just repeatedly send 6 frames worth of data to the poor client, they can't process that (plus they will be really angry with their ISP bills!)

So, we need optimizations, and the biggest one is that we don't need *all* of the 6 faces (the back for example seems unnecessary) and we note that even if we are wrong, since movement is tied to user reaction (which is slow!, at least compared to the FPS), we won't end up being *too wrong*.

![[Pasted image 20240304033057.png|300]]

So, we don't need this whole thing! Just the center and some amount of view around it (marked in a red rectangle), this is reasonable enough to send to the client to fix the rendering.

>[!NOTE] RTT Estimation?
>The paper does not seem to discuss how the RTT is estimated, which is really weird. 
>Here, the computation on the server is heavily correlated with the RTT, so a big spike in the RTT, even for a small moment, can result in a really big computation spike on the server side.
>
>I guess it might be worth investigating how they did this? Since this is a pretty unique case, it is rare for computation and latency to be so correlated on a distributed system.

### Impulsive Events

There are events which are inherently asynchronous, which means that the user history may not reflect their likelihood in the slightest (like firing a gun, a user might fire a gun not because they have to, but because they like the sound!).

The only way to fully account for these is to assume they can happen for all prediction steps. So if we look ahead $T$ steps into the future, then we should produce results for $2^T$ executions **For Each Event!**

We were reluctant to send 6 frames, sending an exponentially scaling number of frames seems ludicrous by comparison! So of course, we need another optimization.
Here this is easy, since while these are a lot of frames, they are different iterations of *the same frame*, so there is a lot of redundancy among this big set of frames. 
We can take advantage of this and compress it down, and this results in a much tamer bandwidth utilization.
## Speculation

There is a general assumption throughput all of this discussion that input is Markovian, an assumption which is natural but not immediately obvious. The paper does go out of its way to mention that it is not a bad assumption and more sophisticated assumptions did not yield satisfactory results to be worth the effort.

We still however, need to account for two big differences in our speculation. Some outcomes are continuous and some are discrete:

**Continuous Input:**
- Use a model to predict input 
- Add redundancy to output (the cube!) to account for miss-predictions
- Implement logic for clients to patch up things if they are wrong

**Discrete Inputs:**
- Execute all possible inputs and generate results for client
- Let the client pick the one that matches their input

This kind of generalizes all that we mentioned above, where the client view was an example of a continuous input, while impulsive events are discrete ones.
## Safety Checks

The system employs both *Checkpointing* and *Rollback Support*.
Why do we need it?
- If we guess wrong too may times, it can have a cascading effect, so we need to support rollbacks.
- We also need checkpoints, since if we don't use them, we might have to re-compute from scratch which can take a really long time.

## Evaluation

Evaluation was done by inviting actual users to come and play with this feature set in place. They would essentially ask for the users opinion on how the games played before and after the feature was installed.

One general rule of thumb for evaluating user inputs is that you should also pair it with actual system output, since a user might either lie or just give a bad score for the sake of it!

For all the evaluations:
- *Standard Thin Client:* A client using a cloud service without Outatime
- *Standard Thick Client*: A client executing the game program locally, no network used at all

### QoE and User Performance Scores

![[Pasted image 20240221152459.png|300]]

This is the user QoE score. As you can see:
- Both cloud clients suffer with increase latency as expected
- *Outatime* slows down the decline, it cannot eliminate it

For more objective results:

![[Pasted image 20240221152724.png|700]]

As you can see:
- *Outatime* does improve the survivability of the player (this is evaluated on Doom 3), since it allows them to cope with latency much better in face of asynchronous events (i.e. a monster jumping on you when you were not expecting it)
- *Outatime* also provides enough buffer against network latency so that players can still *play the game*, even if the quality degrades.

### Frame Time Measurements

![[Pasted image 20240221153659.png|400]]

Here, the *Very Thick Client* is running a version of the client program that is running at a much higher fidelity, to the point that it is actually straining the typical user hardware. 

- The Thick client is still reliably working as expected.
- *Outatime* is shifted to the right compared to the thick client, which makes sense, because there is an actual network with a latency in between the client and server
- *Outatime* performs much better than the standard Thin client, which exceeds the 32 milli-seconds deadline for 30 FPS performance
- *Outatime* is much more stable than the Very Thick client, which has a very variable tail because of the strain the rendering puts on the client hardware, whereas the servers have all the hardware in the world to spare.

As for the frame rate itself:

![[Pasted image 20240221154117.png|400]]

As you can see, *Outatime* is actually **degrading** the FPS compared to the normal client, but that is okay, since as long as the inputs are delivered in time and we don't hit below 30FPS, the user wil be happy!
