# Link Budgets

First, let's start with a block diagram:

![[Pasted image 20230912181254.png]]

In brief:
- When a signal is sent by an antenna, it is concentrated in a particular direction, and as such, it is expressed as a **gain** factor. This depends on antenna itself (an isotropic antenna, one that sends data in all directions, has less gain with fixed power).
  Same goes for the receiver
- When passing through the channel, the signal is attenuated, losing power as it moves forward.
- Both when passing the channel, and when passing through equipment, noise is added to the signal.

Here, the signal "Attenuation" is the main thing to worry about. We can measure the other factors (even noise), but attenuation depends on many things (like distance at the very least).

Two questions you will usually see in this scenario:
1. How much transmit power do you need to get a good signal?
2. What is the maximum distance you can cover before signal quality becomes too low?

## Characterizing Noise

Without getting to bugged down about noise, it is usually "given" to us in most engineering contexts (there is a lot of empirical measurement, but we should not care about that). The 3 ways you see this is one of the following at least:
1. As spectral density in *W/Hz*
2. As Noise Temperature in *K*
3. As Noise Factor (or Noise Figure if it's in dB)

In general:
$$
N_s = kT_s = kF_sT_0
$$
Where $k$ is Boltzmann's constant.

>[!IMPORTANT]
>The noise spectral density of AWGN with the above definition is measure to be -174 dBm/Hz. You should remember this number.

One insight from the simple fact that more bandwidth will have more noise accumulated in it, is that to achieve high data rate through high bandwidth alone, you would have to deal with more noise, and given a fixed power, **will cover less distance!** This is beyond the fact that high frequencies might not propagate well to begin with ...

>[!EXAMPLE]
>A bandwidth of 20 MHz, would be $73\;dBHz$, and would accumulate $-174 + 73 = -101\;dBm/Hz$.

>[!IMPORTANT]
>If you have forgotten to much from Comm. Sys. , remember to convince yourself that no amplifier can improve SNR, it will only worsen it.
>
>While you are at it, also convince yourself that if given a cascade of amplifiers, the amplifier with higher gain and lower noise figure should come first to optimize the final noise figure.

## Friis's Law

To characterize attenuation, let us start with the easiest one, an isotropic antenna (which you can't build, but it's fun to reason about). The antenna distributes the signal uniformly over a sphere, so power drops with the square of distance.
$$
L = (\frac{\lambda}{4\pi d})^2
$$
So our previous equation gives us:
$$
P_{RX} = P_{TX}.G_{TX}.G_{RX}.L
$$
One way that we usually reason about this, is again like many things, in dB scale. Normalizing the frequency by 1 GHz, you would have:
$$
10.Log_{10}\;L = 10.Log_{10}\;{c^2} - 10.Log_{10}\; (4\pi d f)^2 = -32 -20.Log\;\frac{f}{1 Ghz} - 20 Log\;\frac{d}{1m}
$$
Or in other words, with a carrier of 1 GHz, you have -32 dB path loss in 1 meter.
So just by going away by a whole kilometer, you would lose 60 dB of power!

![[Pasted image 20230912203240.png]]

The free space path loss is a nice starting point, but models need to be tuned to each environment. The easiest way is a version of this same model, but with extra parameters (essentially, we allow the slope to be less than -20 dB per distance in dB).

## EIRP

Again , putting everything in dB scale, it is customary to hide antenna gains and just report the effective output power of the antenna. We call this Effective Isotropic Radiated Power or EIRP, which can be calculated as just:
$$
EIRP(dB) := P_{TX}(dB) + G_{TX}(dB)
$$
If we are directly referring to the input power of the antenna before the gain, we usually refer to it as the **Conductive Power** of the antenna.

## Fading Margin

We have discussed fading a little bit, we know that it is an effect **on top of** path loss. So, we need to come up with some characterization of it.
This is hard, since fading varies on a much smaller scale compared to the previous parameters, so we need some statistical characterization of it that tells us how it increase the attenuation *Most Of The Time*. This value is referred to as the **Fading Margin**, and would be essentially defined as "The amount of extra attenuation that you should overcome in order to have good quality most of the time".

We have not yet characterized "good signal" and "most of the time", we need to start thinking about it.

>[!NOTE]
>Usually, small-scale and large-scale fading are considered separately, precisely because they have different margins. So you would have $F_{SSF}$ and $F_{LSF}$. To get the whole margin in dB, just add the two!

With all the dB additions, it might be worth reminding you this!

![[Pasted image 20230912204759.png]]
## Characterizing Good Signal

The easiest way to characterize "Good Signal" is using the SNR. A good signal has a SNR larger than a threshold. That works fine, but it really isn't useful for today's systems.
There are many reason, but one at least is "Adaptive Modulation", where an SNR might be good under a modulation, but unacceptable under another one.

For now through, we stick to this definition.
The problem of keeping the received SNR higher than some threshold is referred to as "Link Budget" problem.

>[!EXAMPLE]
>We want to transmit a signal to down-link, carried over $2\;GHz$.
>The signal is carried in a bandwidth of $20\;MHz$. The transmit and receive gains are $10\;dB$ and $-2\;dB$.
>The signal traverses a Combiner before being transmitted that adds $2\;dB$ of noise power, and the receiver as a whole has a noise figure of $7\;dB$.
>With a fading margin of $12\;dB$ and a transmission power of $40\;dBm$, what is the received power?
>
>To solve this, first we start with the input power to the channel. 
>$$
>40\;dBm + 10\;dBm - 2\;dBm = 48\;dBm
>$$
>The white noise will kill $-101\;dBm$ of this and the receiver gain takes away another 2. The noise figure further degrades the signal. Adding the Fading Margins would give us a **Required** attenuation of -82. So as long as the attenuation of the path is not more than $46 - (-82) = 128$, we are fine.
>
>With FSPL, the loss is:
>$$
>32 + 20 log (2GHz/1GHz) + 20 log (d/1m) = 32 + 6 + 20 log(d/1m)
>$$
>So for 100 meters, the attenuation would be $32 + 6 + 40 = 78$.


