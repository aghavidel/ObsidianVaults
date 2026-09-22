We are grateful to the reviewers for their time and valuable feedback. We will address the questions raised below:

# Q1
We detail the mathematics of this early on in Appendix B.
`OnlineTE` assumes strong convexity (we add small regularizers `f_ek` to make sure of this), as such the optimal solution is unique (if it exists). It is easy to see that this solution cannot have a loop.
As for routing, each switch uses source routing for edge-based allocation, and since each switch only allocates for demands for which it is the source of, the routing will be loop-free as well [1].

# Q2
To our knowledge, there are no publicly available TM datasets with a size comparable to KDL. TEAL and HARP use matrices gathered from private WAN networks, unavailable to us. It is even less likely that such data would be available in 20 second intervals (be default, 5 minutes).

TODO: Discuss fairness to other baselines next ...

# Q3
Our evaluations briefly discuss this. There are two cases: 
    1. For demand shifts, `OnlineTE` can safely warm-start from the last intermediate solution if it fails to converge in time. In figure 9, this occurs at t=300 seconds (note the lack of a flat interval before t=300)
    2. For link failures, edge-based `OnlineTE` recalculates the SVD and reprojects its previous solution and then warm-starts from the result. Path-based `OnlineTE` uses the FRR solution to warm-start.
       This is shown in figure 11, where at t=200 a failure happens when `OnlineTE` still has not converged for the failure at t=180. Convergence takes noticeably longer but still happens.

TODO: Discuss if we can get an estimate of how many loop iterations we'll need ...

# Q4
We compare DeDe to `OnlineTE` in table 1.
DeDe employs ADMM with a decomposition that is very different from the Sharing problem that `OnlineTE` uses. The Sharing decomposition generates much smaller messages and is more suited for communication over geographically disparate nodes; DeDe uses distributed shared memory with Ray to communicate among the processes.
DeDe is unable to sense demands directly or program the allocations in-network, as the individual processes are not attached to the switches. These same considerations apply to the work of Chen et. al. as well, which considers specifically the cloud setting.

TODO: How to approach the point about learning-based methods?

# Q5
This is an excellent point. We acknowledge that our current sparse formulation indeed does not penalize links with high latency.
One effective fix for this issue is to assign weights proportional to the latency of each link and penalize assignments with higher weights.
With our current sparse solver, this translates succinctly into just adjusting the value of the regularization coefficient (`kappa`) with the link weights. This does not change the structure of the solver at all.
We will make sure to incorporate this change into the final version of `OnlineTE`.

## TODO: Address individual reviews if we have space ...

[1] Krentsel, Alexander and Saran, Nitika and Koley, Bikash and Mandal, Subhasree and Narayanan, Ashok and Ratnasamy, Sylvia and Al-Shabibi, Ali and Shaikh, Anees and Shakir, Rob and Singla, Ankit and Weatherspoon, Hakim: A Decentralized SDN Architecture for the WAN
