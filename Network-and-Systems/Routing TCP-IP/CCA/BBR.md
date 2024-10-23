# Intro.

This note describes BBR. A congestion control algorithm described [here](https://queue.acm.org/detail.cfm?id=3022184)

BBR, like many other CCAs, takes issue with how TCP uses packet loss as a measure of congestion, since the two do not necessarily reflect the same states in a network. A well tuned network will not exhibit much packet loss, but that by no means indicates that there are not parts of the network that are congested, or there are not queues that sit only inches away from filling up completely.

## Reviewing TCP's Path Model

One thing that [[TCP Congestion Control]] definitely got right, was how a point to point protocol would see the connection path. The path, no matter how many links are involved with it, is essentially a *single link* in both directions, with the same RTT as the original ones, but the bandwidth of the bottleneck link (the link with minimum bandwidth).

This provides a simple model for all paths considerable for such a protocol, these paths can be modeled with `RTprop` and `BtlBw` and `BtlBuf` parameters, where the first is the equivalent to the RTT, the second is bottleneck bandwidth and the third is the amount of buffer available to the bottleneck.

This creates the following dynamic, where depending on the amount of data inflight.

### For The Delivery Rate

Depending on the number of inflight packets:
- The amount is always bounded by `BtlBw`.
- The rate increases linearly with the size of the window, which is correlated with the amount of inflight data at each moment.

### For The Round-Trip Time

For the RTT we have:
- The round trip time is always bounded from *below* by `RTprop`.
- For very high amounts of inflight data, the amount of bottleneck buffer determines RTT, since when the buffer becomes full, we can only send data that was stored in the buffer, not data that actually makes it into the link.
- For anything in between, the RTT increases linearly with the bottleneck rate.

Combining the two dynamics, we end up with the following:

![[Pasted image 20221006004254.png|500]]

Since normal TCP CCA uses packet loss as a measure of congestion, the aggregate of all conversations over a link end up filling the link completely, taking all available space for the links bandwidth-delay product. This put's these protocols right at knee of the upper figure, where even slight deviations of the operating result in packet loss.

But if efficiency is what we want, the optimum operating point is actually the knee in the lower plot, since it *maximizes delivery rate* and *minimizes RTT*.

>[!FAQ]- Why Was This Not A Big Deal Before?
>Well, mostly because buffers were expensive, and thus the linear region in the upper plot ended up being quite small and the two operating points would end up on the same place roughly speaking.
>But now, buffers are much cheaper, and are orders of magnitudes larger than the BDP of our cross-continental links, the two OPs are much further apart now because of this.

This ideal operating point is sometimes referred to as *Kleinrock's Operating Point* due to Leonard Kleinrock showing that it optimized bandwidth and delay both for individual connections and for the network as a whole.

Sadly however, while Kleinrock's result was still true, Jeffrey Jaffer showed that [it was impossible to converge to this point using any distributed algorithm](https://ieeexplore.ieee.org/document/1095152).

The result rests on a fundamental ambiguity about measurements of RTTs in a network, since an increase in RTT can be either increase in path length, decrease in bandwidth or queuing delays caused by another competing traffic. But still, that does not mean that we wouldn't be able to get pretty *close* to it either way, and that's what BBR (Bottleneck Bandwidth Round-trip) protocol is trying to do.

## Path Characterization

To get near the optimal operating point, we need to have BDP amounts of inflight packets at exactly `BtlBw` rate. So we need to have an estimator for both of these. Since BDP equals the product of bandwidth and RTT, we end up estimating `BtlBw` and RTT similar to original TCP CCAs.

Let's start with the RTT. `RTprop` that we mentioned before are *physical* characteristics, they do not vary that much over the timescale of individual connections, hence we can use the following model for RTTs:
$$
RTT_t = RTprop_t + \eta_t \quad \quad (\eta_t \geq 0)
$$
Where $\eta_t$ is some non-negative random variable that represents the noise of the RTT (it can encompass all sorts of weird behaviors from ACK aggregations, traffic shapers and what not).

Now, the values of the propagation time can be assumed to be constant for the most part during our sampling interval. Now, since we know that this noise is non-negative, an unbiased estimator over samples taken within a window of length $W_R$ for the value of $RTprop$ would be:
$$
\hat{RTprop} = RTprop + \min(\eta_t) = \min(RTT_t) \quad \forall t\in [T - W_R, T]
$$
Where $T$ is the current time.

For the bandwidth though, the only estimators that we have is the rate of data delivery (i.e. the time it takes for a whole window to be acknowledged). These values are always upper bounded by `BtlBw` (that is just how capacity works). Now, we can estimate the delivery time by recording both packet departure times and the amount of data that has been acknowledged. TCP already does the former, the latter is usually something that the application layer holds instead, but from now on we will move it to the transport layer.

Having these values, the rate of data delivery at time $t$, written as $R_t$ can be estimated by dividing the total amount of acknowledged packets by the total delivery time. Knowing that this is upper bounded by `BtlBw`, an estimator for it would be to keep the maximum instead:
$$
\hat{BtlBw} = \max(R_t)\quad \forall t \in [T - W_B, T]
$$
But now, we come back to the inherent *ambiguity* that these two values exhibit. If one wishes to measure `BtlBw`, one must first push the link to the point that it overfills, and then measure the amount of data delivered. On the other hand, this act creates queues that obscure the true value of `RTprop` by introducing a noise that is not necessarily uncorrelated with the input traffic.

Same goes the other way around, accurate measurements of path length require that we stay far away from the maximum bandwidth to make sure that any queue on path quickly dissipates, obviously this cannot be used to give us an accurate measurement of the bandwidth.

## Implementation

Each ACK provides new RTT and average delivery rate measurements, updating `BtlBw` and `RTprop`. If we were to speak this with simple code, then:

```python
def onAck(packet):
  rtt = time.now() - packet.sendtime  
  update_min_filter(RTpropFilter, rtt)
  
  delivered += packet.size  
  delivered_time = time.now()
  deliveryRate = (delivered - packet.delivered) / 
				 (delivered_time - packet.delivered_time)  
				 
  if (deliveryRate > BtlBwFilter.currentMax || ! packet.app_limited):
     update_max_filter(BtlBwFilter, deliveryRate)  
     
  if (app_limited_until > 0):
     app_limited_until = app_limited_until - packet.size

def send(packet):
  bdp = BtlBwFilter.currentMax × RTpropFilter.currentMin
  if (inflight >= cwnd_gain × bdp):
     # wait for ack or retransmission timeout  
     return
     
  if (time.now() >= nextSendTime)  
     packet = nextPacketToSend()  
     
     if (! packet):
        app_limited_until = inflight  
        return
        
     packet.app_limited = (app_limited_until > 0)  
     packet.sendtime = time.now()
     packet.delivered = delivered  
     packet.delivered_time = delivered_time  
     ship(packet)  
     nextSendTime = 
	     time.now() + 
	     packet.size / (pacing_gain × BtlBwFilter.currentMax)
	     	     
  timerCallbackAt(send, nextSendTime)
```




