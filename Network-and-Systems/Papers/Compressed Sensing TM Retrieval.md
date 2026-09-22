**Paper:** [Spatio-Temporal Compressive Sensing and Internet Traffic Matrices](https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=6058636)

This paper concerns retrieval of full Traffic Matrices (TMs) when a good chunk of the entries are missing. This is basically just a matrix completion problem, which has been beaten to death in the past decade.

However, there are a couple of considerations that make the problem quite a bit different in the context of network TMs. This paper outlines these concerns, and proposes a method, Sparsity Regularized Matrix Factorization (SRMF) to implement it.

# Intro.
This paper proposes a compressed sensing methodology for interpolating an entire TM by only a few measurements. This builds on quite a bit of prior work, that established pretty well that TMs can be in some cases, approximated with low-rank creatures.

Compressed sensing (CS) thus becomes attractive, since it is probably the best tool to take advantage of this property, while also allowing us to make do with as little measurements as possible.

However, that observation by itself is too little to allow for normal compressed sensing methods to work. In its most basic form, the CS type problem that we would like to solve, starts with:
$$
\mathcal{A}(X) = B
$$
Where $\mathcal{A}$ is some (known) linear operator, $X$ is a matrix that we want to estimate and $B$ is a matrix of measured values. With the domain knowledge that $X$ is probably very sparse/low-rank, CS allows us to recover such estimates of $X$ when it is very under-determined.

For this to work however, some assumptions are needed (at least for traditional CS):
- $X$ is *exactly* low-rank or sparse. Some level of noise can cause problems.
- Missing entries of $B$, are independent of each other. That is not the case in real networks, where due to say business constraints, entire, correlated groups of values might be missing.
- The operator $\mathcal{A}$ satisfies the Restricted Isometry Property (RIP)
  
>[!REMINDER] RIP
>Given a matrix $A$ and some positive $\delta$, RIP asserts that for all $x$:
>$$
>(1 - \delta) \|x\|_2^2 \leq \|Ax\|_2^2 \leq (1 + \delta) \|x\|_2^2
>$$

None of these requirements hold by themselves, so any method used needs to be slightly adapted to handle such cases.
