This note sources from the following two papers:
- [IQC For Optimization Algorithms](https://laurentlessard.com/public/siopt16_iqcopt.pdf)
- [ADMM as a Dynamical System]([A General Analysis of the Convergence of ADMM](https://proceedings.mlr.press/v37/nishihara15.pdf))
Both based on the good work done by Laurent Lessard at UC Berkeley.

# Background (Linear) on Dynamic Systems

A discrete linear dynamical system, with input $u_k \in \mathbb{R}^d$ and output $y_k \in \mathbb{R}^d$ can be specified like:
$$
\begin{aligned}
\xi_{k+1} &:= A \xi_k + Bu_k \\
y_k &:= C \xi_k + Du_k
\end{aligned}
$$
Where $\xi_k \in \mathbb{R}^m$ is some internal state vector. We may also add a feedback loop into the system by specifying a function like $\phi(.)$ that asserts:
$$
u_k := \phi(y_k)
$$
Now, assume that for some function $f$ we let $\phi(.) := \nabla f(.)$, we get the following system:
$$
\begin{aligned}
\xi_{k+1} &:= A \xi_k + B~\nabla f(y_k) \\
y_k &:= C \xi_k + D~\nabla f(y_k) \\
\end{aligned}
$$
Now, a first order method for solving an optimization problem, generally only relies on the gradient as a measure of where to go for the next step, and thus is always able to be transformed into the format that we described above.

For example, GD with fixed step size $\rho$ can be found by renaming $\xi_k \to x_k$ and setting:
$$
A := I_{d\times d}\quad B:= -\rho I_{d\times d}\quad C:= I_{d\times d}\quad D:= 0_{d\times d}
$$
We will focus on discussing how reasoning about dynamical systems, can help reason about the *convergence* of (first order) optimization algorithms. In Control Theory, we discuss a general line of analysis collectively called **Stability Analysis** where it is discussed:

- To what value does $\xi$ converge to?
- What is the speed at which we converge over $\xi$

