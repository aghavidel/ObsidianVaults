**Paper Link:** https://www.usenix.org/system/files/nsdi23-perry.pdf
**Conference:** NSDI 2023 - Fall

# Abstract and Intro.
The paper aims to solve TE optimization on WANs only using <u>historical data</u>.
To meet the demand for high traffic and availability, many network providers solve the traffic optimization problem across vast portions of their networks. Since these are complex and often time-sensitive problems, many turn to heuristics or make compromises in order to finish these computation in time with acceptable accuracy.

The problem here is "what is the traffic in the near future?".
There is 2 types of this problem:
1. **Time Sensitive:** Get telemetry data from switches and interpolate for the near future.
	- **Problem:** It's just hard to predict things, as you need to make preparations for when you are wrong.
2. **Bandwidth Hungry:** Deploy agents in traffic sources and relay the demands of each source to a logically centralized provider.
	- **Problem:** 
