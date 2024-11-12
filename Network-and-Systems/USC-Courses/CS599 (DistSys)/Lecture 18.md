(lectures 16 and 17 were Map-Reduce and Spark ...)

# ExCamera

The goal here is some sort of server-less computing.
Until now, basically every system that we have used is supposed to run over a pretty long time interval, and also requires a dedicated server for coordination and computation.

This paper focuses on a different type of computation, in particular, one that is:
- Short lived, and needs to spin up quickly
- Should be billed fine-grained (as opposed to say, hourly ...)

One particularly fine example of this type of computing is with AWS Lambda.
AWS provides an infrastructure where many small Linux containers that can be spawned at will, and execute (ideally) arbitrary code. Lambda is pretty unique since it does actually tick the condition for running arbitrary code.
Besides being able to be spawned very quickly, they are also billed for execution time, thus extremely fine grained, compared to how say, AWS EC2 instances are billed at much coarser grain (1 minute at the time of writing), which means that even if they are executed with the exact same spec, they will cost more, unless it has run for a significant time.

There are a few things with Lambda though:
- Has a timeout on how long you can run it (5 minutes when the paper was written, 15 minutes at the time of writing)
- All workers are behind a NAT, and AWS does not allow port forwarding or hole punching that easily, so you cannot initiate connections to Lambdas from outside, the Lambda has to do it first.
- There is a limit on how many concurrent instances you can have
- Individual Lambdas are not coordinated with each other, thus having dependencies or synchronization requirements can cause problems for correct executions.

ExCamera, the paper in question, developed *mu*, an API that allows  most of the above limitations to be handled.

## mu

The idea is that:
- The user writes arbitrary code and compiles it into a binary
- `mu` then writes a generic function that accepts the above binary as an argument and executes it
- Two static EC2 instances are used to control the Lambdas:
	- A *coordinator* server the submits requests for Lambdas and gets the results back
	- A *rendezvous* server that listens for Lambdas.
		- Each Lambda will initiate a connection to the rendezvous server and keep it open, essentially bypassing the NAT.
		- When needed, the server is used to relay messages between Lambdas, and would allow for executions to be synchronized.
- In order to spawn Lambdas fast, they need to be *warm* (which means that they should be able to receive and accept TLS connections).
  However, you cannot spawn a huge amount of Lambdas at will, since as far as AWS is concerned, it would be indistinguishable from a DoS attack, thus the functions have a rate limit, and so if you want to spawn many functions, you should pace it.
  To solve this, `mu` spawns Lambdas first and keeps them warm. Since they don't execute anything, this won't cost you much. Executions starts the moment that the TLS connection from coordinator to the Lambdas is fully initialized.

![[Pasted image 20241105123422.png|500]]

In the above, the 1 second delay for warm start is completely due to establishing TLS connections to Lambdas from coordinator server.

This is all fine and well, but what to actually use it for?

# Video Processing with `mu`

The main concern of the paper is *interactive video editing*, in particular, we want to be able to easily and interactively, pick a frame in some video at a certain point and just edit away as needed. There  are a few problems here though:
- Videos have a *codec*, that allows them to compress frames.
  To encode, one would pick a time interval and then:
	- Use the first frame of the interval as a *key frame*
	- Store the difference of subsequent frames in the interval (these are called *inter-frames*)
- The decoder would then load from key frames, and rebuild frames by adding inter-frames back to the key frame. The problem of course here is that to edit any frame, you *need* to load the key frame as well, and key frames can be pretty large.

As such, since codecs use temporal correlation to compress videos, if you use a huge amount of key frames to allow interactive loading of frames, you would severely degrade compression efficiency.
As such, decoders and encoders have an internal state, which an only be fully reconstructed when the key frame is available. If we want to be able to decode any frame at will, we need to be able to reconstruct that state immediately.

One codec that is widely used is `VP8`, and it provides a simple API for encoding and decoding 
```
decode(state, frame) -> (state', image)
encode(key_frame, image) -> (state, frame)
rebase(state, image, interframe) -> interframe'
```
In particular, if you invoke the encoder on a list of images, you would get a key frame and a bunch of interframes:
```
encode(images[1:n]) -> (keyframe, interframes[2:n])
```
So the way ExCamera works is like the following:
- The video is encoded on the user side with any codec (again, Google's VP8 for example)
- A bunch of key frames and interframes are sent to the coordinator, and the system would then pass them to a separate Lambda (so each Lambda has a key frame, and a bunch of interframes).
- (*Weird step*), no we actually *decode* the final frame in what the Lambda received using normal VP8, this gives us a state (the first return value in the API spec).
- We pass this to the neighboring Lambda function, and by doing so, we would be able to discard the key frame associated with it, if and only if we can modify the codec so that it would be able to accept an explicit state as an argument as well.

The final thing above is the main issue. By default, the codec never exposes its internal state, and even the `state'` return value is not explicit, we can just inspect the codec data structure to see what it is.

An explicit state passing codec would provide something like the following:
```
encode-given-state(state, image, quality) -> interframe
rebase(state, image, interframe) -> interframe'
```
Here:
- The encoder accepts a state, and using that state, it generates an interframe for an image with some specified quality.
- Using the new interframe, we can rebase all ....

(READ THE PAPER, I DID NOT UNDERSTAND ANYTHING THAT SEO JIN SAID!)

