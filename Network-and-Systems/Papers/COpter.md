**Paper:** COpter: Efficient Large-Scale Resource-Allocation via Continual Optimization
**Link:** [COpter: Efficient Large-Scale Resource-Allocation via Continual Optimization | Proceedings of the ACM SIGOPS 31st Symposium on Operating Systems Principles](https://dl.acm.org/doi/10.1145/3731569.3764846)

This paper presents *Continual Optimizer (COpter)*, a cluster resource scheduler designed to very quickly solve problems by continually solving problems using their previous solution (obviously, this only works if the the input changed slightly, which is a point we'll discuss later).

# Introduction

This paper is mostly oriented towards cluster scheduling. While TE problems (at least MLU and Max Flow) and cluster scheduling are both LPs on paper, they have different practical considerations, so a few things discussed in this paper really only apply to one case and not the other.

But for both of these settings, the traditional methods of solving them are basically the same:
- The problems are solved within *rounds*. Each around a few minutes (1 minute for cluster scheduling, 5 minutes for WAN TE). When a round starts, a new set of inputs are given to us and we must solve the problem.
- Problems for each round are solved from scratch. Virtually no state is preserved between rounds within the solver.
It is the second part that we want to emphasize. There are two aspects here that we should take issue:
- As hinted above, the inputs do not change fundamentally between rounds. The heavy hitter definitely take more than a few rounds to change significantly. As such, the majority of the output solution also doesn't change by much.
  The figure below supports this observation for the case of cluster scheduling.
  ![[Pasted image 20251106172204.png]]
  In the figure above, a layered figure is shown describing the ratio of variable changes, additions and deletions (note that in Cluster Scheduling, demands come and go, unlike TE where every node communicates with another, and we just change the demand value itself).
  Take note that the majority of the variables have not changed.
- On another hand, solving problems from scratch isn't scalable. Even meeting the 60 second deadline by itself is pretty difficult. Take this result for example:
  ![[Pasted image 20251106172537.png]]
  In the above, $\text{LP}^*$ is a general LP solver (e.g. Gurobi), and POP-n are partitioned solutions (where we partition the problem into some subproblems and solve them independently and then concatenate, paying at times significant price in terms of optimality).
- There are algorithmic challenges in using previous solutions. This method for solving things is usually referred to as "warm-starting". For stable problems, where a small change in the input translates to a small change in the optimal solution, some algorithms may not even be able to efficiently make use of extra information.
  Generic methods for solving LPs generally include a few algorithms:
	- **Simplex:** Outputs very clean and nice solution, and each iteration is very cheap, but since it jumps from vertex to vertex, a small interior point distance in the $L_2$ sense does not translate to a short path on the polyhedron (think of a very small Klee-Minty cube)
	- **IPMs (e.g. Barrier):** The outputs are quite messy, necessitating a few Simplex iterations to make them pretty, but if you don't care about that, they are very easy to use and much quicker than Simplex because of multiprocessing. However, they are extremely difficult to warm-start due to numerical issues as well as requiring the iterates to be well-centered (in fact, most solvers ignore initial solutions for these and just solve from scratch).
	- [[PDLP]]: This is somewhat new, and considering it's a first-order method, you *can* actually warm-start it and is very efficient, but not enough (this paper does not mention them, but while it is a pretty substantial improvement over IPMs, it is still not enough by itself).

# Cluster-Scheduling as LP

In Cluster-Scheduling (CS), we receive as input a bunch of jobs that request some amount of resources (in this paper, it is GPUs). Let the vector $G \in \mathbb{R}^m$ denote this (so we have $m$ jobs). When job $j$ is assigned its $g_j$ GPU resources, it has a known throughput of $c_j$, so consider the vector $c \in \mathbb{R}^m$ to encode this. 
If we employ a binary vector $x \in \{0, 1\}^m$, the total throughput would be $\langle c, x \rangle$, which we would want to maximize.
The constraint here is that we have limited resources, say $N$, which we can encode by $\langle G, x \rangle \leq N$, thus we get the problem:
$$
\begin{alignat*}{2}
\text{min}_x& \quad -\langle c, x \rangle \\
\text{s.t.}&\quad \langle G, x \rangle \leq N \\
&\quad x \in \{0, 1\}^m
\end{alignat*}
$$
We switched to minimization to keep this LP in standard form (i.e. $Ax \leq b$ for constraints for minimization, $Ax \geq b$ for maximization).
The second constraint makes this problem a Mixed-Integer LP (MILP), which adds substantial difficulty, but there are post-processing methods that can operate on the LP *relaxation* of the algorithm to push an optimal non-integer solution towards the optimal integer solution.
As such, for now, we discuss only the LP relaxation, the integer part we discuss later.

## Computational Challenges 

With state-of-the-art generic LP/MILP solvers, some challenges exist when solving the problem above, some of which specifically become a point of pain because we solve these **repeatedly** ...
- As stated above, warm-starting is very non-trivial with Simplex/Barrier, but PDLP can actually use that.
- *Compiling* and *building* the optimization repeatedly, is costly. Generic solvers usually keep some data-structure that is necessary between iterations to quickly progress (in Simplex, it is the basis, and in IPMs, it is usually some factorization of the constraint matrix). There is no simple way of evolving these data-structures between rounds, so we would have to compute them from scratch, which takes precious time.
- MILPs are combinatorial creatures. The usual way to solve them is to first solve the LP relaxation, collect variables that have large integer infeasibilities, and then recursively solve an LP where we add a constraint that forces them to the two nearest integer values (i.e. if $x_j = 4.5$, solve again by constraining $x_j = 4$ and $x_j = 5$).
  Obviously, this adds a huge overhead ...
The way that this paper tackles these issues is as follows:
- For warm-starting, this paper makes use of first-order methods, in particular [[Proximal Point Algorithms]] (PPA). These methods can be trivially warm-started by just setting the initial iterate at the desired point (ADMM also enjoys this flexibility).
- For compiling the problem, this paper uses a *differential interface*, which we will discuss
- For handling the MILP issue, this paper notes that integer infeasibilities are rare if good heuristics are used, thus we really don't have to solve too many LPs repeatedly.

# Continually Solving LPs

To somewhat formalize the notion of LPs that have basically the same optimal solution, the paper defines *Slowly Evolving LPs*. Intuitively, an LP is slowly evolving if over $T$ rounds of solution:
- The $L_2$ norm of the optimal solution concentrates around some value
- The problem dimensions remain roughly the same
For some LP in standard form, with the constraint matrix $A \in \mathbb{R}^{m \times n}$, one can define the problem dimension to be $\eta = \max (m, n)$ and over $T$ rounds, we can just pick the maximum of this.

>[!NOTE] Slowly Evolving LP
>For a sequence of LPs $\text{LP}_t$ for $t \in \{1, 2, ..., T\}$ in standard form, we say that they evolve slowly if parameters $0 < \alpha, \beta \ll 1$ and $0 < B$ exist such that:
> - $\sum_{t=2}^{T}  |~ \eta_t - \eta_{t-1} ~| = O(T^\alpha \eta_{\text{max}})$
> - $\sum_{t=2}^{T} \| x^*_t - x^*_{t-1} \|_2^2 = O(T^\beta B)$
> In the definitions above, the vectors $x^*_t$ are padded to up to $\eta_{\text{max}}$ when needed.


