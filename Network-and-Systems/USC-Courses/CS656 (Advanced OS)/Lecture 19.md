# Energy vs Fidelity

The paper for this lecture is quite different compared with other papers that we discussed until now. This is one of the first paper that raises concern about how applications utilize energy and how to adapt to that.

This is a problem even today, quite simply because energy efficiency and battery technology is outpaced by how rich and diverse our software and hardware needs are becoming. We have all sorts of different things to add to mobile devices and that puts strain on the batteries and power sources.

>[!EXAMPLE]
>A traditional approach to energy saving is to modify the hardware fidelity. For example, most CPUs come with a voltage adjustment setting that can lower their clock cycle, which in turn decreases power usage when not much computation is needed.
>
>On the other hand, we can identify idle devices using help from the OS, and then put them on a low power setting or just turn them off for some time.
>The problem with these approaches is that *when* the time comes for us to return to the previous state, it is difficult to do that (these devices need some warmup time usually), so what else can we do?

The main idea of the paper is to adjust application quality of service. For example:
- Deliver lower bitrate to reduce energy consumption on the network interfaces
- Use lower quality models in ML tasks
- Use sparse samples of data from disk to lower disk IO to answer queries approximately

## Odyssey Design

The paper designs a user space process for modifying the fidelity of applications dynamically. From a high level view, odyssey looks like this:

![[Pasted image 20240325142233.png|500]]

The system implements a kernel level interceptor that catches calls from the user space process, and makes an upcall into the specific application that needs to be tuned. 
There are 4 categories of applications being evaluated:
- A video player
- A web browser
- A map viewer
- A speech recognizer

### Video Player

![[Pasted image 20240325142452.png|500]]

Above is the result of Odyssey optimizations for the video player application.
The optimizations being done are:
- Hardware only: Lower CPU cycle and NIC bit rate
- Premiere B and C: Lower video frame rate and resolutions
- Reduced window: Smaller GUI window

The power categories are:
- Idle: Just the OS and its own processes and hardware support. Always the majority of the power
- `Xanim`: The video application itself
- `X Server`: The GUI server
- `Odyssey`: The Odyssey application
- `WaveLAN`: The network interface of `Xanim`
- `Kernel`: The kernel processes

As you can see from the picture above, each single optimization by itself has marginal impact, its the combination of multiple optimizations that actually makes a tangible change.

Some takeaways:
- With lower video quality, network energy drops because of lossy compression and less data being sent
- With a smaller window, the `X Server` energy usage drops heavily
- Idle energy is always the biggest chunk

### Speech Recognizer

![[Pasted image 20240325144130.png|500]]

Here, `Janus` is the application itself, that converts an utterance into text. The model of this application can be run on two levels of fidelity. It is also possible to have a remote instance of it on a server from the network and use that to just get the result. 
A hybrid approach uses both the remote and local instance, the bulk of the computation is done on the server and the local instance just speeds things up by doing some pre-processing on the data.

The takeaways are mostly:
- Running `Janus` locally is expensive, hardware power management like before helps, but not very much
- Using a reduced model will reduce computation overhead and lower energy usage
- Using remote instance helps a lot, but it also wastes time being idle while waiting for a response
- Having reduced models in the remote scenario helps by reducing the amount of data sent
- Hybrid approach helps, since again, it lowers the amount of data sent

### Web Server

![[Pasted image 20240325144755.png|500]]

The usage categories that are new are:
- `Netscape`: The browser app
- `Proxy`: The proxy application used to access the internet

The main reason that idle is such a huge part of the energy usage is because the user has to input image requests manually, so it takes a long time!

This is the main reason that in this setting, the benefits are very marginal and small.

## Other Approaches

One thing to note is that `idle` is always the big monkey, and the heaviest part of that monkey is the display. The display is a big resource hog, and thus, the authors proposed an approach called *Zoned Backlighting*. 

The approach is, divide the display into $n$ zones, and then deduce which zones are currently in the focus of the user, and if they are not, dim them to save power.
An extreme version of this is used for TVs, where *every single pixel* can be a zone!

## Effects of Concurrency

For now, we just looked into optimizing a single application, but that quite simply is not realistic. We have  many applications running at the same time at any instance. 

Consider we have applications $A_1$ and $A_2$ that consume energy $E_1$ and $E_2$ by themselves. If we run them at the same time, the energy would be $E$, now:
- It is possible that $E < E_1 + E_2$, for example if we can sequence execution of one application when another is idle we get lower energy than the sum!
- On the other hand, it is also possible that $E > E_1 + E_2$ if running both applications causes resource contention

Using the previous 4 applications as an example, we run the Speech Recognition, Map Viewer and Browser application at the same time and then run the video application with them, and we get the following:

![[Pasted image 20240325151549.png|500]]

The main takeaway is that hardware only management becomes less attractive as the applications become more diverse, fidelity adjustment is necessary for such a scenario.

To tune fidelity adjustment, it is also beneficial to be *goal-directed*, letting the user specify how long they want the battery to last and then use that number to adjust the fidelity. Of course, to make that adjustment, we need to see if the goal is feasible in the first place, and to do that, we can run all applications on the lowest fidelity, and then see if the power is low enough to support the user demand, and if it is not, then tough luck!

On the other hand, there is a more nuanced optimization, **maximize the fidelity while minimizing changes in fidelity**. This actually leads to a pretty stable and nice user experience, since changes in fidelity are much more notable (e.g. repeated changes in video quality).

So how to do this?
1. Measure how much energy we have
2. Estimate future demand given the current fidelity level
3. Ask apps to change fidelity if demand is less than supply

The first one is easy, measure the battery life time while being conservative. Step 2 is hard, but usually we just use past observations and interpolate that to a longer time. Again, we use some heuristics to make that a bit more safe.

For step 3, we might have to ask the user to specify which applications can be slowed down.