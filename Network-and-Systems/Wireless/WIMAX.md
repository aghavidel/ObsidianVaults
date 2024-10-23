# Intro.

The *Wireless Interoperability of Microwave Access*, or just *WiMAX*, also known as the 802.16 IEEE standard, is a wireless communication technology adopted in many mobile and base stations in 2000s, until it was largely set aside for the more superior [[LTE]] for the 4th generation of networking standards.

WiMAX facilitated the evolution of many technologies (notably [[MIMO]]) but most of it's solutions came too late and offered too little. It saw a rerelease as WiMAX 2, but at that time, LTE-A was already established for the most part.

Without delving into details, WiMAX is essentially Ethernet without wires, built to be used as the infrastructure that would support large networks of CPEs or provide connection between larger wired networks in difficult to reach areas. To this end, it was designed to have better range compared to it's other competitors (most notably [[Wi-Fi]]).

It was for the most part, designed to be used by larger network providers, with much lesser concern for power consumption, as it was not really designed to be used for local networks (which was a wise decision, since Wi-Fi already had the edge for LANs, being much easier to setup and much less taxing on the batteries of the devices it was used on). 

Instead of small PCB antennas, WiMAX would use larger base stations, and so it was only natural that it paved the way for MIMO in LTE and other technologies. To make it more appropriate for this purpose, it was designed to have much longer range (a few 10s of miles, compared to the few hundred feet that Wi-Fi could work with) and provided higher data rates per channel.

It is also worth mentioning that WiMAX was designed to support many more subscribers, allowing hundreds of CPEs to connect, and letting subscribers and CPEs have many-to-one mappings (in contrast to the one-to-one mappings of CPEs and subscribers in Wi-Fi). As you can see, WiMAX was a very good choice for large networks and MANs, at least until LTE came.

## Basics

WiMAX builds on top of previous [[OFDM]] technologies for modulation. We will not get into the nitty-gritties of how the [[Physical]] layer is managed, instead we mostly focus on the [[MAC]] layer and scheduling. 

Two definitions first:
- **Subscriber Station**: Or simply SS, is the equipment that provide connection between the subscriber (the user device) and the Base Station. 
- **Base Station**: Or BS, is the main point of access, providing connections to the subscriber stations and governing the message passing and scheduling. It can consist either of simple WiMAX electronics and WiMAX towers, that use fixed dish antennas. Each tower operates on a *cell*, that can theoretically cover a 50 km radius, but practical consideration in urban environments usually cut it down to 10 km.
  The stations will implement OFDM and the WiMAX specific MAC layer for uplink and downlink. We'll discuss the MAC layer shortly. Stations can use *backhaul* connections, which can be high speed wired links (like a T3 link) to connect multiple stations, effectively allowing for roaming users to pass between cells. 


### WiMAX Physical Layer

The physical layer will implement OFDM, with convolutional error correcting codes at various different rates. Both BS and SS will provide feedback to each other about the state of their channels using Channel Quality Indicators (CQIs), similar to LTE, the standard dictates what modulation scheme and coding rate will be used based on the CQIs.


When a SS attempts to connect to a BS, multiple phases of communication ensue, we will go over each phase briefly below:

## Initialization:

Initialization involves the exchange of messages called MAC Management Messages, or `MAC-MGMT` in two distinct phases, *Synchronization* and *Ranging*.

### Synchronization

Like all wireless protocols, synchronization involves the process of checking which channel can the BS and SS use for communication, and then to check if the channel characteristics are acceptable for the SS.

