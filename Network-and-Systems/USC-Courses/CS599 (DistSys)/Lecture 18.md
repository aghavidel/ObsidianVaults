# ExCamera

The goal here is some sort of server-less computing.
Until now, basically every system that we have used is supposed to run over a pretty long time interval, and also requires a dedicated server for coordination and computation.

This paper focuses on a different type of computation, in particular, one that is:
- Short lived, and needs to spin up quickly
- Should be billed fine-grained (as opposed to say, hourly ...)

One particularly fine example of this type of computing is with **AWS Lambda**.
AWS provides an infrastructure where many small Linux containers that can be spawned at will, and execute (ideally) arbitrary code. Lambda is pretty unique since it does actually tick the condition for running arbitrary code.
Besides being able to be spawned very quickly, they are also billed for execution time, thus extremely fine grained, compared to how say, AWS EC2 instances are billed at much coarser grain (1 minute at the time of writing), which means that even if they are executed with the exact same spec, they will cost more, unless it has run for a significant time.

There is also the problem of actually setting up the system (i.e. the VMs) in a general case. If it was easy to set up such a thing, we could have used Spark and just destroyed every parallelizable problem with sheer brute force. However, computers cost money, and when you have to pay for VMs every minute, from a purely economical point of view, solving problems in 1 micro-second is no different to solving them in 59.99 seconds.

## Background About AWS Lambda

There are a few things with Lambda though:
- Has a timeout on how long you can run it (5 minutes when the paper was written, 15 minutes at the time of writing)
- All workers are behind a NAT, and AWS does not allow port forwarding or hole punching that easily, so you cannot initiate connections to Lambdas from outside, the Lambda has to do it first.
- There is a limit on how many concurrent instances you can have
- Individual Lambdas are not coordinated with each other, thus having dependencies or synchronization requirements can cause problems for correct executions.
- Further complicating the above, each Lambda has only a single thread of execution (it is literarily just a function call).

Further more. each Lambda container has a *Cold-Start* phase. Meaning that when it is first spawned, the function call takes much longer to finish as the container is just being configured on demand, but later calls would be much faster.

ExCamera, the paper in question, developed *mu*, an API that allows  most of the above limitations to be handled.

## mu

The idea is that:
- The user writes arbitrary code and compiles it into a binary
- `mu` then writes a generic function that accepts the above binary as an argument and executes it. This kills two birds with one stone:
	- Having a generic function that handles everything, means that we can reuse the same Lambda container for future calls if we are quick enough, and this means that we can skip the cold-start phase of a call to a Lambda function.
	- Ship the code as an argument, instead of having to spin up a new Lambda each time (kind of cheating, but it is admittedly quite clever).
- Two static EC2 instances are used to control the Lambdas:
	- A *coordinator* server that submits requests for Lambdas and gets the results back
	- A *rendezvous* server that listens for Lambdas.
		- Each Lambda will initiate a connection to the rendezvous server and keep it open, essentially bypassing the NAT.
		- When needed, the server is used to relay messages between Lambdas, and would allow for executions to be synchronized.

In order to spawn Lambdas fast, they need to be *warm* (which means that they should be able to receive and accept TLS connections).

However, you cannot spawn a huge amount of Lambdas at will, since as far as AWS is concerned, it would be indistinguishable from a DoS attack, thus the functions have a rate limit, and so if you want to spawn many functions, you should pace it.

To solve this, `mu` spawns Lambdas first and keeps them warm. Since they don't execute anything, this won't cost you much. Executions starts the moment that the TLS connection from coordinator to the Lambdas is fully initialized.

![[Pasted image 20241105123422.png|500]]

In the above, the 1 second delay for warm start is completely due to establishing TLS connections to Lambdas from coordinator server. As for how the warm-start is done, essentially the same computations are invoked repeatedly (3 times according to the paper), thus skipping cold start.

>[!REMINDER] The Limits Of Coordination For Parallelizable Tasks
>Every task usually has a fixed computational cost in general that cannot be parallelized. These things cause huge parallelization of tasks to have sudden diminishing returns, until they would no longer be worth the cost of operating them. This is usually referred to as *Amdahl's Law*, but on paper it is quite intuitive.
>What bares mentioning though, is that this really only becomes the bottleneck if tasks are *embarrassingly parallelizable*, meaning that not only can they be chopped up as needed, but they also barely need coordination.
>This would be something like Spark, which scales quite well. If we do however need coordination (as we do in this paper for example), then coordination becomes the main bottleneck at scale, to the point that the fixed parts of the computation lose any relevance.
>
>![[Pasted image 20241202153218.png|500]]

# Video Processing with `mu`

The main concern of the paper is *interactive video editing*, in particular, we want to be able to easily and interactively, pick a frame in some video at a certain point and just edit away as needed. There  are a few problems here though:
- Videos have a *codec*, that allows them to compress frames.
  To encode, one would pick a time interval and then:
	- Use the first frame of the interval as a *key frame*
	- Store the difference of subsequent frames in the interval (these are called *inter-frames*)
- The decoder would then load from key frames, and rebuild frames by adding inter-frames back to the key frame. The problem of course here is that to edit any frame, you *need* to load the key frame as well, and key frames can be pretty large.

>[!EXAMPLE] Example of Key-Frame Size
>As mentioned in the paper, consider a 4K video. A single frame of this video would be around 11 MB, and when encoding with a bit rate of 15 Mbps with `VP8`, a key frame is around 1 MB, and with good, static shots, the inter-frames hover around 10-30 KB.
>As such, we need to be economical with how many key-frames we put into a video.

As per the description above, chunks of a video that are between key-frames, are in essence, independent from a compression point of view. As such, they can be effectively parallelized with a simple scatter-gather method (which is what ExCamera tries to do as well).

As such, decoders and encoders have an internal state, which an only be fully reconstructed when the key frame is available. If we want to be able to decode any frame at will, we need to be able to reconstruct that state immediately.

>[!IMPORTANT]
>There is a tradeoff here between latency and compressions:
> - Less key frames $\rightarrow$ More inter-frames $\rightarrow$ more compression
> - More key frames $\rightarrow$ Shorter interval between independent chunks $\rightarrow$ lower latency
> Thus, naive, massive parallelization would have to rely on having a key-frame per thread, which degrades compression.
## Primer on `VP8`

>[!FAQ] *Frame* vs *Image*
>This naming is important:
>- **Image**: The raw, untouched instance of a video. These are the big 11 MB arrays of a  4K video.
>- **Frame**: A compressed *Image*, an implementation dependent entity, that looks nothing more than just a normal array.

To compress an image, the easiest way is to keep track of pixel values, and create a codec that maps more frequent values to smaller codes. That is exactly what `VP8/9` do. This is what they refer to as a *probability model*, in essence, nothing more than a mapping from values to frequency and code, and vice-versa. This is the main component of the running state of a decoder. For other reasons, `VP8` also needs a set of 3 reference images to be included as well (these are raw images, and contain the decoded output of previous decoder invocations, so the system has some temporal memory).

Thus, a *state*, is in essence, nothing but:
$$
\text{state} := \langle \mathbb{P}, (I_{-2}, I_{-1}, I_{0})\rangle
$$
Now, let us look into `VP8` API; it is very bare-bones by design:
```
decode(state, frame) -> (state', image)          // Output a new image
encode(key_frame, image) -> (state, frame)       // Consume image, spit frame
```
In particular, if you invoke the encoder on a list of images, you would get a key frame and a bunch of interframes:
```
encode(images[1:n]) -> (keyframe, interframes[2:n])
```
As for where the state comes from, it is usually initiated using the key-frame.
```
decode(*anything*, key_frame) -> (state, image)
```
The key-frame has metadata within it that `decode` considers, which allows to distinguish key-frame from other things, and execute differently. As for what a frame in general actually contains, it is usually of the form:
$$
\text{inter-frame} := \langle M, V, R \rangle
$$
Where:
- $M$ is the *Prediction Modes*.
  These are pointers to groups of 4 by 4 pixels that are visually similar to the data in the image corresponding to this frame. The data is pulled either from somewhere within this image, or one of the 3 reference images. The decoder can use this to patch up the image into a fairly complete state, but still with missing pieces.
- $V$ is a set of *Motion Vectors*.
  It describes *where* reconstructed patches generated from $M$ actually end up (so if visually similar materials move between frames, we can still use them).
- $R$ is the residue. The diff of the reconstructed image and the original one.
  If the encoder is efficient, we can hope that this part ends up quite small, but no real guarantees exist on it.
  The residue might have a hard limit on size, meaning that we would have to rely more on predictions, which can potentially degrade image quality.

## Explicit State-Passing

For ExCamera, the following routine was also implemented:
```
encode_given_state(state, image, quality) -> interframe
```
The above, receives a `state` as an explicit state to use from encoding, and a raw image as `image`, and some quality metric (think of it like PSNR).
The function then use the supplied state as if the key-frame was already provided to it, and encodes the image as usual.

So what's important here? Well, note that this works on *any* image if the state is available. A state is just a set of 3 reference images and not too large of probability table, much smaller than a key-frame. Thus, this effectively allows one to replace key-frames with inter-frames.

However, there is a catch. Inter-frames of course depend on the state, even if they predict the exact same picture. So, if we just yank the inter-frame that represents a key-frame from a previous chunk, and give it as is to a decoder, it would either fail or produce garbage. Thus, we need some routine that can make inter-frames *look* like they were generated with an arbitrary state.

For this, we have a `rebase` function:
```
rebase(state, image, interframe) -> interframe'
```
The idea is:
- Unpack `interframe`, and read $M$ and $V$.
- *Apply* $M$ and $V$ to the provided `state`. This makes a temporary new state that is compatible with the provided inter-frame.
- The above won't be lossless if the original state prediction was not that good. Thus, we need to recalculate the residue. This is why the `image` parameter exists. It holds the raw image that corresponds to this inter-frame, allowing for compensation for any loss of data in this process.

The benefit here, is that we skip recalculating $M$ and $V$, which is good, since they are the slowest part of encoding an image. Producing the new residue on the other hand can be done very fast, so `rebase` is pretty efficient.

## How ExCamera Encoder Works

The main insight here is to parallelize calculations of $M$ and $V$ for inter-frame generation, and only use serial processes to for building $R$. Calculating modes and motion vectors is the *Slow* part, and $R$ is the *fast* one (hence the name of the paper).

So the way ExCamera works is like the following:
- **`(INITIATION)`** First, `mu` spins up $x$ Lambdas, each acting as a completely independent thread that would not have to suffer from cold-start. This also assigns an index to each thread, meaning that each thread except the final one has a *next-thread* attached to it.
- **`( PARALLEL )`** Chop up the video into fine pieces. For example, a quarter of a second, which considering most films are a steady 24 frames per second, ends with 6 frames per chunk. This means that each Lambda that is under the control of `mu` would have to download $N$ frames of a video (in this example, we set $N$ to be 6).
  Thus, we would have scatter $N.x$ frames over all Lambdas at the end of this step.
- **`( PARALLEL )`** Invoke Google's VP8/9 encoder (i.e. `vpxenc`) in parallel on each chunk. The output contains a key-frame and $N-1$ inter-frames. The size of the output is dominated by the key-frame which is Megabytes of size, whereas the inter-frame are only a few tens of a kilobyte.
- **`( PARALLEL )`** Invoke *ExCamera*'s decoder serially, over all the images in each thread (so a total of $N$ invocations). This produces a final `state` value. Call the *Rendezvous* server to inform the next-thread instance about what the value of this state is.
- **`( PARALLEL )`** This step depends on thread index:
	- *Index 0 (i.e. the first thread):* You are done! Upload your output (which has a single key-frame and $N-1$ inter-frames) and exit.
	- *Other threads:* Invoke *ExCamera's* `encode_given_state` using:
		- `state` as gathered from the Rendezvous server
		- `image` as the first image of your chunk
	 This produces an inter-frame, replace your key-frame with this and discard the key-frame. This is the *Slow* part, but responds well to parallelization.
- **`(  SERIAL  )`** Thread index 1 onwards will now invoke `rebase` on each of their remaining $N-1$ interframes. They will use:
	- `state` as the one that was gathered from the Rendezvous server
	- `image` as their raw image corresponding to the inter-frame
	- `interframe` as the inter-frame that `vpxenc` generated
  All other threads will follow, this as well. When a thread is done rebasing, the output result is uploaded and the thread finally exits.

At the end of this, we would have quickly encoded $N.x$ frames, but only used a single key-frame.

Here is a timeline of how it works:

![[Pasted image 20241203024300.png|600]]
