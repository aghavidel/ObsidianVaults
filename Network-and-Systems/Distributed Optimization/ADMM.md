This is take from the reference [survey of ADMM by Boyd. et.al]([ADMM](https://web.stanford.edu/~boyd/admm.html)).
We will follow the survey's structure closely, as it is a very good read. This writing is more an attempt to convince the author that he understood the material!

# Background

We start with some background, some of which is known well, but it wouldn't hurt to recap. We defer the discussion of ADMM until later.

## Dual Ascent Methods

One of the oldest methods of solving optimizations with equality constraints is *Lagrange Multipliers*, or the more fancy (and appropriate) name of *Dual Ascent*.

In particular, given the following problem:
$$
\begin{align}
	\text{minimize}&\quad f(x)\\
	\text{s.t.}&\quad Ax = b\\
	&\quad x\in \mathbb{R}^n
\end{align}
$$
Given the assumption that $f: \mathbb{R} \rightarrow \mathbb{R}$ is convex, we can do dual ascent by constructing the Lagrangian $L(x, y) := f(x) + y^T(Ax - b)$ and the dual function $g(y)$ as:
$$
\begin{align}
g(y) := \inf_x L(x, y) \equiv -f^*(-A^Ty) - b^Ty
\end{align}
$$
where $f^*$ is the Convex-Conjugate of $f$.

The dual problem would be to just maximize $g(y)$.
This can be used to solve for the optimal value of the original problem, if strong duality holds and there is a single maximizer for $g$, then we can get the minimizing $x$ for the original problem.

The paper uses the notation