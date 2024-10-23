This is the first lecture of EE 535 (Wireless Communications), instructed by professor [Andreas F. Molisch](https://wides.usc.edu/founder.html) (yes, THAT Molisch, the one who wrote the big book!) at the University of Southern California.

Some insights:
- The main application of Wireless Communications is Smartphones. But that is changing, it is becoming an infrastructure for many different things.
- It is easy to think that Wireless Communications is a mature technology, that has had its time in the wild for enough. That might be true for again the same smartphone usage, but it is most definitely NOT TRUE for all the other new stuff that we are trying to do.
  In fact, the data rates are going down fast with the increasing demand.
- Wireless Communication is a gigantic part of technology infrastructure, it is NOT going anywhere for at least a few decades. 
- Wireless systems and Wireless Propagation Channels are extremely non-linear systems, and we are still not really good at controlling them without significant constraints on their dynamics, so a lot of interesting tools will need to be developed for effective use of them.

For this first lecture, we discuss:
1. The types of service for aw wireless system
2. The parameters that describe a wireless system
3. The tradeoff between each parameter
4. What is the fundamentally hard problem in wireless systems

# Types of Service

## Cellular Phones

![[Pasted image 20230911001931.png]]

In the simplest sense, the main thing that defines a wireless system and differentiates it, is *mobility*, the fact that User Equipment can move between Base Stations without loss of connection. Even this simple model comes in different flavors:
- We have **Satellite Systems**, where cells are very large and base stations are expensive
- There is also **Trunked Radio** where some UEs are grouped together and can only communicate with each other.

## Wireless LAN

![[Pasted image 20230911002345.png]]

These are much easier to make, since there is much less mobility:
1. You do not need a Mobile Switching Center (MSC)
2. You can work with just one Base Station (BS)

## Personal Area Network

The distance between BS and UE is small. An example would be Bluetooth. The significant factor is that:
1. There is barely any mobility
2. There is no coordination between modules and controllers

You can already see that one thing that happens with systems that have mobility, is *interference*, this is going to be one of the main challenges of wireless communication.

Beyond that, there is also the challenge of not really knowing *where* the UE is. To compare this:
1. If you have a dish for a satellite, it can be directed in one direction with high spectral efficiency.
2. When you have a radio station, you might need significant power to just achieve not terrible spectral efficiency.

## Broadcast TV

![[Pasted image 20230911002908.png]]

The funny thing about these systems is that **they should not care about the number of users**. They just send out data, no matter whether 1 person is listening or a million.

To put this in perspective, when you hear people say that "we have 100 GB rate!", what that usually means is "we have 100 GB rate, when you are alone, standing next to the base station and the stars have aligned", add 1 more person, and the rate drops of to maybe half! Huge difference, which broadcast systems cannot tolerate.

## Ad-Hoc Networks

![[Pasted image 20230911003156.png]]

This is weird, since there is **no infrastructure**. For example, in the event that USC's network goes down entirely (say in a natural disaster), you can enable a feature called "Wi-Fi direct", that allows cellphones to talk to each other with no coordination.

The main thing here is that despite the total lack of coordination, these systems are cheap, since:
1. There is no BS, so you don't have to worry about powering it up, cooling it, and etc.
2. There is no hefty requirements, since people are grouped close

On the other hand, almost always, these systems turn out to be spectrally inefficient. Why? Well because each node is small little device with a small little battery, there is just enough power to break through spectral inefficiencies. 

# Wireless System Descriptors

Also called Key Performance Indicators (KPIs). 

## Data Rate

The big one, how much data you transmit in a unit of time. A big range for different systems:
- **Sensor Nets and IoT:** Less than 1 Kbit/s, the coordinator node might need 10 Mbit/s
- **Speech Comm.:** Depending on the vocoder, 5-64 Kbit/s
- **Elementary Data Service**: At least in the US, you should be able to at least get 10-100 Kbit/s
- **Computer Peripherals:** You need at least 1 Mbit/s to achieve good synchronization
- **Web Browsing and Better Data Services:** Peak data speeds around 1 to 100 Mbit/s
- **Video Streaming and Conferencing:** 0.2-20 Mbit/s, heavily depends on the video compression. Images are large, you need a factor of 100 in compression to make them feasible to send.
- **Personal Area Networks:** 100 Mbit/s - 10 Gbit/s, depends on the application. If you want VR that does not make your stomach upset, you need really high data rates!

## Range, and Number of Nodes

-  **Body Area Networks:** At most 1 m range, one user with less than like, 10 nodes
- **Personal Area Networks:** Like, 10 m range, again less than 10 nodes
- **Wireless LAN:** You need to support less than 100 m easily, and a few hundred nodes. This is where the number of nodes becomes really important.
	- You need specific protocols to make this work. Even with a beefy access point, a bad protocol implementation can result with laughably bad performance.
- **Cellular Networks:** range of 100 m to like 10 KM. In each cell you might have 10-50 users active at each time, but the number of **potential users** can be hundreds of thousands.
- **Fixed Wireless Access:** (i.e. a base station that serves a particular area, like a remote village). Range up to 10s of KM, with similar requirements as a Cellular net.
- **Satellite Systems:** Thousands of KMs, thousands of potential users. You need dedicate stations just to communicate with these things!

### Range vs Rate Tradeoff

![[Pasted image 20230911010213.png]]

We do not have Eierlegende Wollmilchsau (Egg laying wool milk pig!).

## Mobility

- **Fixed Device:** Stays in one location, but may have temporal variance due to objects around it. You can base the design of your system on the fact that it won't move for years.
- **Nomadic Device:** The UE is mostly stationary, but moves in the span of a few hours. You should have coverage for many areas, but each area can assume that you are not moving around too much. An example of this would be *Wireless LAN*.
- **Low Mobility:** Pedestrian speed. People are not assumed to stay in one place, so be prepared for them to bounce between stations.
- **High Speed:** Like a cellphone in a car!
- **Really High Speed:** Like a cellphone in a train or a plane!

### Rate vs Mobility Tradeoff

![[Pasted image 20230911011303.png]]

## Energy Consumption

- **(Chargeable) Battery:** A cellphone or a Nomadic device would need about 16 - 24 hours per charge. You can't put too much weight on a battery.
- **One-Way Battery:** A non-rechargeable battery that works for like 10 years. You need this for sensors, and these design require ***extremely low*** power.
	- Curiously, this is a lot less about designing circuits, and much more about designing protocols.
- **Power Main:** You need this for beefy things, like a base station. One unfortunate consequence of consuming a large amount of power, is producing a large amount of heat! A lot of operation cost would now be created just to handle this (like just think of the poor Air Conditioner that needs to cool this thing down!)

## Spectrum Usage

There are 3 main types of spectrum usage:
- **Dedicated to specific Service and Operator:** Basically, operator de jour comes in, putting a huge amount of money on the right to use a spectrum for cellular connection.
- **Dedicated to specific Service:** Like for Wi-Fi, no one owns it, but you need to operate in it each time you are dealing with Wi-Fi.
- **Free:** Free for all. 

For the last two cases, you have a lot of constraint on the amount of power you can use. If you scream too loudly, you will annoy the neighbors. There is however, a few tricks that you can use to scream a bit louder than you would expect. Again, there is a lot of protocol design involved here. The main thing here is that we should know what type of interference to expect. If we don't know what type of interference to expect, you should be able to tolerate unforeseen drop of quality. 

## KPIs of 5G

![[Pasted image 20230911012532.png]]

Context:
- IMT-Advanced is 4G/LTE
- IMT-2020 is 5G

>[!NOTE]
>Perhaps a big one for 5G is **Latency**, which was quite ignored in the previous generations. 
>Why? Well, at least one reason is self-driving cars. You CAN NOT tolerate for a car to be told to stop, only for it to kill someone and then say "who? me?!".
>
>Another one is **Connection Density.** Why would this matter?
>Well, the problem is that each connection always has some amount of overhead. This overhead adds up per UE, so if you have thousands of them bundled up under one BS, you would end up wasting a huge amount of resource just because of that!

>[!QUESTION]- How would you achieve very high rate for high densities?
>Just build more Base Stations. One around every corner.
>The problem then becomes how to justify this. If we need a Base Station every where, then just use a wired connection or a fiber!

# Major Challenge of Wireless Communication

- **Broadcast Effect:** You scream, everyone hears you!
- **Multipath Propagation:** You send one thing, you receive it from 5 directions!
- **Spectrum Limitations:** You can't scream to loud, or too frequent.
- **Limited Energy:** You ay not be able to fix things with raw power.
- **User Mobility:** Things are not time-invariant.

## Broadcast Effect

We send signals in all directions many times. 
- This is good in some sense, since it makes the transmission process unrelated to the number of users.
- This is bad in another sense, since you are annoying the neighbors who were not even talking to you in the first place!

## Multipath Propagation

![[Pasted image 20230911014213.png]]

Again, there is two sides to this:
- The first thing is that we have "Small Scale Fading". Which means that there is now the chance of destructive interference between the signals coming from different paths. The main effect of this is that you can have vastly different connection quality by just moving an inch around a point. This is why we call it "Small Scale" ...
   
![[Pasted image 20230911014428.png]]

- There is also "Large Scale Fading", where you have huge power variation because of shadows cast by objects. This is not really a consequence of multi-pathing, but we should consider it.
  
![[Pasted image 20230911014558.png]]

Why this gives engineers nightmares?
- For one, there has always been a catch-all solution to improving communication quality. Scream louder!
  Mathematically, this comes from the somewhat general result that the Bit-Error-Rate is a Q function of SNR. So if SNR is higher, you get better communication (though with diminishing returns).
  This is NOT true in wireless systems, and the main reason for that is Fading:
	  For small scale fading, increasing power may not even matter if you are in a fading dip!
	  For large scale fading, increasing power won't help much
  In general, fading causes the relation between SNR and BER to be linear at best and uncorrelated at worst.
- Aa a direct result of the above, you NEED statistical model of mobility and channel, since it becomes too difficult to create deterministic models, and the consequences of a wrong model are way too severe. 

## Inter-Symbol Interference (ISI)

Since different multi-path components have different phases, you don't get just an attenuation, but a whole dispersion. In particular, what this means is that your previously pure and clean impulse response, now has a lot small spikes in it, and they add up to make things ugly!

![[Pasted image 20230911015342.png]]

If you still remember basic comm. systems, then you know we solve this with equalizers and nice thresholds, but even that turns out to be way more difficult. We'll discuss this at length later.

## Spectrum Assignment

You should learn not to look at spectrum ranges as just numbers, or homogenous slots in an array. Each spectrum range offers different physical properties, and as such, might be better suited for different applications. As a somewhat accurate rule of thumb:
1. Lower frequencies are absorbed less by every-day materials, so they travel further, but by definition we just cannot get much rate our of them.
2. Higher frequencies facilitate higher data rates, but they are finicky, and get lost in the environment much easier.

## Frequency Reuse

Since spectrum is highly limited, we need to learn to reuse it. One main insight about this, is the fact that frequencies are NOT LOCKED to the particular area that they are deployed (this is what makes Cellular Communication, well, Cellular).

## Energy Limitation

We have already discussed not being able to scream too much, but there is also design impacts to consider. For example, this has ramifications for:
- Signal processing
- Modulation
- Sensitive RX if you are using low power
- Provision for a "Sleep Mode" in certain situations

## User Mobility

Mobility means that your channel is going to change, whether in a cell, or between cells. As such, you cannot build an equalizer with a hundred knobs, tune it, and then call it a day. We are way beyond LTI channels at this point, the estimation needs to be repeated, and with high efficiency (like in the scale of a few milliseconds).
This process is called "Channel Estimation", and is fundamental to the majority of signal processing that goes into Wireless Communication.

**END OF LECTURE**
