
# Jupiter Rising (cont.)

We finished discussing FH 1.1, and we noted that it was a nightmare to actually cable, so what now?

## Designs Before Jupiter
### Watchtower

The main development since FH 1.1 was an advancement in Merchant Silicon, allowing them to have 16 links of 10 Gbps each! This was a huge step up and potentially meant two things:
- No longer should the fabric bother with 1 Gbps links, so cabling would be less of a pain
- The network could now theoretically be non-blocking!

So we have switch chips that have 16 ports of 10 Gbps, Google designed a line card with 3 chips on the board, and each one would be packaged as a single line card. An array of 8 line cards would then be used to make a single Watchtower chassis.

![[Pasted image 20240214072349.png|300]]

This topology is non-blocking and provides 128 (8 times 16) port switch with 10 Gbps each. Using 64 as uplink and 64 as downlink we can replace the aggregation blocks in FH 1.1 with these.

Google also approached the problem of cabling with much more care this time. The decision was made to make use of optical cables instead (kind of the only logical choice now since everything is running at 10 Gbps).

![[Pasted image 20240214072852.png|600]]

### Saturn

Keeping up with the trend, Saturn was the next iteration of Watchtower with 24 port chips, each at 10 Gbps.

![[Pasted image 20240214073502.png|500]]

Following the same logic as in watchtower, instead with 12 line cards this time, we get $2 \times 12 \times 12 = 288$ port switches per chassis. Saturn, similar to watchtower, also provided non-blocking aggregation blocks.

One of the more subtle changes was in the ToR itself. Google rolled out the Pluto chips, which were switches with 24 ports of 10 Gbps, supporting 20 servers with 4 uplinks, thus giving an oversubscription ratio of 5:1. However, Google left room for other, more bandwidth hungry servers to use an alternative configuration, where the number of servers were lowered to 16 and the remaining 8 ports were bundled to uplink. 
This gives an oversubscription ratio of 2:1, but it also tolerated full bandwidth bursts, since the average bandwidth for each server was 5 Gbps.

## Jupiter

Jupiter was where Google started getting fancy with their data centers. It was at this time where Merchant Silicon chips started to have 40 Gbps ports, and so a big jump in network speed was observed, however, it was very unlikely that any time soon, this 40 Gbps design could be ubiquitous enough to be used everywhere.

Thus, Google made the choice to design Jupiter with multiple networking speeds in mind, extending the limited ideas that they had with the ToR in Saturn. 