This is taken from the reference [survey of ADMM by Boyd. et.al]([ADMM](https://web.stanford.edu/~boyd/admm.html)).
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
g(y) := \inf_x L(x, y) = &\inf_x (f(x) + (A^Ty)^Tx) - y^Tb = \\
&\sup_x ((-A^Ty)^Tx - f(x)) - y^Tb = \\
&f^*(-A^Ty) - y^Tb
\end{align}
$$
where $f^*$ is the Convex-Conjugate of $f$.

The dual problem would be to just maximize $g(y)$.
This can be used to solve for the optimal value of the original problem, if strong duality holds and there is a single maximizer for $g$, then we can get the minimizing $x$ for the original problem.

Most of the time, we assume that strong duality holds, which is not true, but that consideration makes most of the things that we will discuss next moot, so we'll push that aside for now. We'll revisit it once we have laid out all that we need to discuss.

So moving forward, assuming that strong duality holds, the optimal value of $g$ and $f$ would end up being the same. We can recover a dual optimal point $y^*$ from solving the maximization problem on $g(y)$.

If we were able to get a dual optimal point $y^*$ from solving the above problem, then it follows from strong duality that a primal optimal point can be found from $x^* = \text{arg}\min\limits_x L(x, y^*)$. If there are multiple minimizers, then the result need not necessarily be primal feasible, and thus not a minimizer. The optimization *WILL* have a single minimizer though, if $f$ is strongly convex.

*Dual Ascent*, is a method that takes cue from this and solves for $y^*$ with gradient descent over $g(y)$. This is quite easy since the gradient $\nabla g(y) = Ax - b$ is readily available, and thus a single GD step would be:
$$
\begin{align*}
	x^{k+1} &= \arg\min_x L(x, y^k)\\
	y^{k+1} &= y^k + \alpha^k (Ax^{k+1} - b)
\end{align*}
$$
Where the superscript denotes the value for the step, and $\alpha$ is some step size, which we can vary for each step as needed.

This is a simple algorithm, but:
- While it is called *dual ascent*, it does not necessarily *ascend* at each step. Meaning that there are times in general where taking a single step puts us in a worse position than we started. This is heavily dependent on the step size $\alpha$.
- More concerningly, it does not even have to *converge*, in particular, in the case where $f$ is non-zero and affine in any component of $x$ (the case with LPs in particular), the $x$-update step returns gibberish.

However, dual ascent illustrates a very important benefit of really any optimization method, *decomposition*.

### Dual Decomposition

If we are able to chop up the objective $f(x)$ in terms of the components of $x$, like say if we could have $f(x) = \sum\limits_{i=1}^{p}f_i(x_i)$, where $x_i$ are sub-vectors formed from some components of $x$, then we notice that we can also chop up $A$ in a similar manner column-wise, and have it written as $A = [ A_1, ..., A_p ]$, which allows us to write the Lagrangian as:
$$
L(x, y) = \sum\limits_{i=1}^p f_i(x_i) + y^TA_i x_i - \frac{1}{p}y^Tb
$$
And as such, dual ascent can be done in parallel, with $p$ independent problems being solved at once to give the solution to the original problem. In particular, the $x$-minimization steps are done independently, while the dual update step aggregates the result from the updates of all problems.

This is handy, but with all the problems that dual ascent has for convergence, it is more cute than useful. Again, all we need is some part of the objective to be linear in some component of the input, and we are toast.

## Augmented Lagrangian And Method of Multipliers

As far as robustness in terms of convergence is concerned, we can do a bit better. We can convert the original primal problem into:
$$
\begin{align*}
\text{minimize}\; &f(x) + \frac{\rho}{2} ||Ax - b||_2^2\\
\text{s.t.}\; & Ax = b \\
& x \succcurlyeq 0
\end{align*}
$$
Which attains the same optimal as the original and even has the same feasible set. If we look at the Lagrangian for this problem though, we see a nice quadratic term added:
$$
L_\rho(x, y) = f(x) + y^T(Ax - b) + \frac{\rho}{2} ||Ax - b ||_2^2
$$
We call this creature the *augmented* Lagrangian of the original problem, with step size $\rho$. The dual function obtained from this method can be shown to be differentiable under much weaker conditions than for normal dual ascent.

As such, using augmented Lagrangian gives much better convergence. As for how to use this, we need only modify two things in the original dual ascent:

- The $x$ minimization step must now use the augmented Lagrangian. So to calculate $x^{k+1}$, we need to set $x^{k+1} = \arg\min\limits_x L_\rho (x, y^k)$.
- We also need to set the step size explicitly to $\rho$ for the dual update step.

The second constraint is needed for dual feasibility. The reason for that is because if the first step indeed minimizes the Lagrangian, then the gradient must vanish at $x^{k+1}$ and $y^k$.
If we apply this to the Lagrangian:
$$
\begin{align*}
	0 = &\nabla L(x^{k+1}, y^k) = \\
	& \nabla f(x^{k+1}) + A^Ty^k + \rho A^T(Ax^{k+1} - b) =\\
	& \nabla f(x^{k+1}) + A^T (y^k + \rho (Ax^{k+1} - b))
\end{align*}
$$
And use an arbitrary step size of $u$ and let $y^{k+1} = y^k + u (Ax^{k+1} - b)$, we see that if $u \neq \rho$, we might fail the dual feasibility condition of $\nabla f(x^*) + A^Ty^* = 0$; thus we choose to let $u = \rho$ to get dual feasibility.

This augmented Lagrangian method, confusingly called the *Method of Multipliers*, converges much better because of the quadratic term, but for that exact same reason, it is NOT decomposable. If we expand the quadratic term:
$$
\begin{align}
||Ax - b||_2^2 &= (Ax-b)^T(Ax-b) \\
&= \bigg(x^TA^TAx - 2x^TA^Tb + ||b||_2^2 \bigg)
\end{align}
$$
We see that we are at the mercy of $A^TA$ for decomposability. In particular, if it is say, block diagonal, then we would get decomposition with respect to its partitions, but otherwise, there is nothing we can do about it.

This is a big hit, meaning that we have sacrificed our ability to chop up the problem into smaller bits, for stronger convergence globally.

ADMM, in part, is an answer to this conundrum.

# ADMM

ADMM hopes to give some decomposability back, while also still have the strong convergence that we get with the augmented Lagrangian. To do this, for now, we focus on problems that have the following structure:
$$
\begin{align*}
	\text{minimize}\; & f(x) + g(z) \\
	\text{s.t.}\; & Ax + Bz = c \\
	& x \in \mathbb{R}^n, z \in \mathbb{R}^m, c \in \mathbb{R}^p\\
	& A \in \mathbb{R}^{n\times p} \\
	& B \in \mathbb{R}^{m \times p}
\end{align*}
$$
This form seems weird, but it is quite general. There are many tricks that can be employed to write a problem like the above (we'll see them in [[#Applications]]).
For now, if we actually want to solve the above, we can write the Augmented Lagrangian again:
$$
L_\rho (x, z, y) = f(x) + g(z) + y^T(Ax + Bz - c) + (\rho/2) ||Ax + Bz - c ||_2^2
$$
And now, we can just solve it with dual ascent, BUT, for ADMM, we do something a little different. In particular, we *split* the updates for $x$ and $z$:
$$
\begin{align*}
	x^{k+1} &\gets \arg\min\limits_x L_{\rho} (x, z^k, y^k) \\
	z^{k+1} &\gets \arg\min\limits_z L_{\rho} (x^{k+1}, z, y^k) \\
	y^{k+1} &\gets y^k + \rho(Ax^{k+1} + Bz^{k+1} - c)
\end{align*}
$$
We refer to these steps as the $x$-update, $z$-update and dual update respectively.

Take note that when going from $x$-update to $z$-update, **the other variable is kept fixed**. This is the main benefit of ADMM. In a normal method of multipliers step, we would have to do a joint minimization on both $x$ and $z$, but here, we don't have to, and the good thing is that we can instead perform minimizations on smaller problems and then combine them with the dual update step.

As you can see, the degree of decomposability here is not as good as we had. The first two steps need to follow each other directly, and the quadratic term is still a problem. *However*, the fact that the updates are not done jointly, opens the way to all sorts of trickery that we can use to split the quadratic term. It is still hard, but at least it is not *impossible*.

With the above in mind though, one might be tempted, to change the second step to:
$$
z^{k+1} \gets \arg\min\limits_z L_{\rho} (x^k, z, y^k)
$$
This makes the two updates truly parallel, meaning that we can do both at the same time and then combine them in the dual step. <u>This is tempting, but is best avoided</u>. The algorithm need not exhibit good convergence in this case, and it is important to consider this, since in general, **ADMM is good for converging quick to things that are good enough, but takes longer to become optimal**, we should probably not trade convergence with scalability too much.

## Scaled Form

ADMM can be written somewhat differently. In particular, we can formulate it in terms of the *residual*, i.e. the primal feasibility gap of $r = Ax + Bz - c$. Further, if we define $u := \frac{y}{\rho}$, we will get the following form for the augmented Lagrangian:
$$
\begin{align*}
L_\rho(x, z, u) = &\quad f(x) + g(z) + \rho u^Tr + (\rho/2)||r||_2^2\\
&\quad f(x) + g(z) + (\rho/2) \bigg[
	||r||_2^2 + 2u^Tr
\bigg]\\
&\quad f(x) + g(z) + (\rho/2) \bigg[
	||u+r||_2^2 - ||u||_2^2
\bigg]
\end{align*}
$$
Where $u$ is our *scaled* dual variable. While this form may look fussy, it looks pretty nice when we apply it to our update steps. In particular, since the dual variable, and hence $u$, are kept fixed during minimizations of the Lagrangian, we can just do away with the lone square term for $u$ and be left with $||u+r||_2^2$.

This gives us:
$$
\begin{align*}
	x^{k+1} &\gets \arg\min\limits_x \Big(  f(x) + (\rho/2) || Ax + Bz^k - c + u^k||_2^2 \Big) \\
	z^{k+1} &\gets \arg\min\limits_z \Big(  g(z) + (\rho/2) || Ax^{k+1} + Bz - c + u^k||_2^2 \Big) \\
	u^{k+1} &\gets u^k + r^{k+1}
\end{align*}
$$
This also gives a nice meaning to the dual update. *It is just the running sum of the residual!*

## Dual Feasibility

The method of multipliers is designed such that each update gives dual feasibility for free. It may not be necessarily obvious that this is the case with ADMM though.

Our functions need not be differentiable, so the primal/dual feasibility conditions become:
$$
\begin{align*}
&\quad Ax^* + Bz^* - c = 0 && \text{(Primal Feasibility)} \\
&\quad 0 \in \partial f(x^*) + A^Ty^* && \text{(Dual Feasibility I)} \\
&\quad 0 \in \partial g(z^*) + B^Ty^* && \text{(Dual Feasibility II)}
\end{align*}
$$
Where $\partial f(.)$ is the sub-gradient operator.

Now, the algorithm asserts that the Lagrangian is minimized at $(x^{k+1}, z^{k+1})$ in the $z$-step, which means that the sub-gradient with respect to $z$ contains a zero at that point.
$$
\begin{align*}
	0 \in &\quad \partial_z \; L_\rho (x^{k+1}, z^{k+1}, y^{k}) \\
	= &\quad \partial g(z^{k+1}) + B^Ty^k + \rho B^T (Ax^{k+1} + Bz^{k+1} - c) \\
	= &\quad \partial g(z^{k+1}) + B^T (y^k + \rho (Ax^{k+1} + Bz^{k+1} - c)) \\
	= &\quad \partial g(z^{k+1}) + B^T y^{k+1} \\
	&\quad \quad \equiv \text{Dual Feasibility II}
\end{align*}
$$
Which is to be expected, since this is basically analogous to how the method of multipliers is dual feasible at each step.

The more interesting thing is why the $x$-update is/isn't dual feasible. Again, by definition, 0 must be a sub-gradient of $L_\rho(x^{k+1}, z^k, y^k)$ with respect to $x$, thus:
$$
\begin{align*}
	0 \in &\quad \partial_x \; L_\rho (x^{k+1}, z^{k}, y^{k}) \\
	= &\quad \partial f(x^{k+1}) + A^Ty^k + \rho A^T (Ax^{k+1} + Bz^{k} - c) \\
	= &\quad \partial f(x^{k+1}) + A^T (y^k + \rho (Ax^{k+1} + Bz^{k} - c))
\end{align*}
$$
Now, let us bring back the definition for the residual. The last expression in parenthesis can be written as:
$$
\rho(Ax^{k+1} + Bz^{k+1} - c + B(z^k - z^{k+1})) = \rho r^{k+1} + \rho B(z^k - z^{k+1})
$$
which once combined with the above, gives:
$$
\begin{align*}
	0 \in &\quad \partial f(x^{k+1}) + A^T (y^k + \rho (Ax^{k+1} + Bz^{k} - c)) \\
	= &\quad \partial f(x^{k+1}) + A^T y^k + \rho A^T (r^{k+1} + B(z^k - z^{k+1})) \\
	= &\quad \partial f(x^{k+1}) + A^T (y^k + \rho r^{k+1}) + \rho A^T B(z^k - z^{k+1}) \\
	= &\quad \partial f(x^{k+1}) +A^T y^{k+1} + \rho A^T B(z^k - z^{k+1})
\end{align*}
$$
Now, we are in a bit of a pickle. The term $\rho A^TB(z^k - z^{k+1})$ is unruly, and we can't say as-is that it will converge to zero so that the above gives dual feasibility. It does however, suggest, that it *should be the case* if ADMM converges at all. In-fact, the proof of convergence for ADMM (which you can find in the appendix of Boyd's paper) relies on showing that it indeed does.

This creature, i.e. the expression $\rho A^TB(z^k - z^{k+1}) = s^{k+1}$ is called the *dual residual* of ADMM, i.e. the thing that prevents it from being fully dual feasible. It is the dual counterpart of the primal residual $r^{k+1}$.

### A Note On Convergence

There is a tradeoff, between primal and dual feasibility, as the algorithm progresses. This is best seen with a choice of $\rho$:

- If it is very large, it penalizes violations of primal feasibility, so the primal residual $r^k$ will quickly converge to a very small value early on.
- If it is small, good enough dual feasibility is possible even with large jumps between the values of $z^k$ at each step.

Note that dual feasibility basically translates to the primal being near some minimum, so essentially we are trading off satisfying the constraints, with making the objective small. There are heuristics however, that suggest that playing with the value of $\rho$ during the initial steps can speed up convergence by making sure that we can lower both residuals quickly.

Now, we should note however, **there is for now, no concrete result that shows what happens to the convergence of ADMM with varying values of step size**, thus if we want to remain robust from a theoretical perspective, we should stop fiddling with $\rho$ after some set number of steps and just keep it constant afterwards.

With this in mind, the paper by Boyd notes some heuristic that comes from another paper, in the form of the following:
$$
\rho^{k+1} = \begin{cases}
\tau^{\text{incr}} \rho^k & \text{if}\quad ||r^k||_2 > \mu ||s^k||_2 \\
\rho_k / \tau^{\text{decr}} & \text{if}\quad ||s_k||_2 > \mu ||r^k||_2 \\
\rho_k & \quad\quad \text{otherwise}
\end{cases}
$$
With a choice of $\tau^{\text{incr}}, \tau^{\text{decr}} > 1$ and some value $\mu > 1$.
The idea is to keep the primal and dual residuals to within a factor of $\mu$ from each other, and to enforce that, we alternate between cutting down or inflating the value of the step size multiplicatively. If this is used, just remember to rescale the scaled dual variable if you are using the scaled version of ADMM.

# Applications

Here, we just throw a bunch of different applications of ADMM that we found related to our work.

## Sharing Problem

This problem is usually presented in un-constrained form as:
$$
\text{minimize}\quad \sum\limits_{i=1}^{N} f_i(x_i) + g(\sum\limits_{i=1}^{N}x_i)
$$
The idea for this problem is to consider each $x_i \in \mathbb{R}^n$ to be a choice from some independent agent $i$, that is trying to minimize some local cost $f_i$, but the problem must also minimize some global cost $g(.)$, which considers the choice of all agents concurrently.
The above is the simplest form of such a problem, with the individual choices just being expressed linearly as the sum of all of the agent choices.

Now, an ADMM solution for this needs to introduce a pairing to $x_i$, like $z_i \in \mathbb{R}^n$:
$$
\begin{align}
\text{minimize} &\quad \sum\limits_{i=1}^{N} f_i(x_i) + g(\sum\limits_{i=1}^{N}z_i) \\
\text{s.t.} &\quad x_i - z_i = 0, \quad i = 1, 2, ..., N
\end{align}
$$
Note the exchange with $x_i$ in the argument of $g(.)$.

Now, if we write the scaled update scheme literarily, we'll get:
$$
\begin{align}
x_i^{k+1} & \gets \arg\min\limits_{x_i} \bigg(
	f_i(x_i) + (\rho/2) ||x_i - z_i^k + u_i^k||_2^2
\bigg) \\
z^{k+1} & \gets \arg\min\limits_{z} \bigg(
	g(\sum\limits_i z_i) + (\rho/2) \sum_i ||x^{k+1}_i - z_i + u_i^k||_2^2
\bigg) \\
u_i^{k+1} & \gets u_i^k + (x_i^{k+1} - z_i^{k+1})
\end{align}
$$
The first and final step is naturally decomposed among individual components, but the second step is a big mess. It requires one to solve an optimization over $Nn$ variables, which can quickly get out of hand.

So here is the idea, let $\bar{z} := \frac{1}{N} \sum\limits_{i=1}^N z_i$ and $a^{k+1}_i = u^k_i + x^{k+1}_i$ (take note that the dual variable is from the not updated, but $x_i$ is updated). The $z$ update can be written as:
$$
\begin{align}
\text{minimize} &\quad g(N\bar{z}) + (\rho/2) \sum_{i=1}^N ||a_i - z_i||_2^2 \\
\text{s.t.} &\quad \bar{z} = (1/N) \sum_{i=1}^N z_i
\end{align}
$$
The idea is to solve this as a *mini-ADMM*, keeping $\bar{z}$ fixed when we want to optimize $z_i$ itself. Which means that the updates would be:

- Optimize for $\bar{z}$, unconstrained with other components.
- Optimize for $z_i$ individually, unconstrained with $\bar{z}$.
- Dual update in the third step, taking into account what comes from above.

This is simple since if we keep $\bar{z}$ fixed, then we have an analytical solution for all other $z$ components, specifically, we need only have:
$$
z_i \gets a_i + (\bar{z} - \bar{a})
$$
As for $\bar{z}$ itself, we need only solve the following *unconstrained* problem:
$$
\text{minimize} \quad g(N\bar{z}) + (\rho N/2) ||\bar{z} - \bar{a}||_2^2
$$
Now, something funny happens with the dual update. Again, the solution above gives $z_i^{k+1} := a_i^{k+1} + (\bar{z}^{k+1} - \bar{a}^{k+1})$ ; if we plug this into the dual update, we'll get:
$$
\begin{align}
u_i^{k+1} & \gets u_i^k + (x_i^{k+1} - (u_i^k + x_i^{k+1} + (\bar{z}^{k+1} - \bar{a}^{k+1}))) \\
& = \bar{a}^{k+1} - \bar{z}^{k+1} \\
& = \bar{u}^{k} + \bar{x}^{k+1} - \bar{z}^{k+1}
\end{align}
$$
Take note that this applies to **all** components of $u$ equally. Essentially, this means that this problem formulation keeps all components of the dual vector in consensus. We can ab(use) this fact, and just replace the whole thing with a single variable $u \in \mathbb{R}$ !

Armed with this knowledge, we can finally write the complete algorithm. First let us clean up any usage of $a$. Our original $x$ minimizer step refers to $z_i^k$, which would be written as:
$$
\begin{align}
z_i^k &= a_i^k + (\bar{z}^k - \bar{a}^k) \\
&= (u^{k-1} + x_i^k) + (\bar{z}^k - (u^{k-1} + \bar{x}^k)) \\
&= x_i^k + \bar{z}^k - \bar{x}^k
\end{align}
$$
And now, if we plug this into the original formulation and replace the previous $z$ update with the update on $\bar{z}$, we'll get the following:
$$
\begin{align}
x_i^{k+1} & \gets \arg\min\limits_{x_i} \bigg(
	f_i(x_i) + (\rho/2) ||x_i - x_i^k + \bar{x}^k - \bar{z}^k + u_i^k||_2^2
\bigg) \\
\bar{z}^{k+1} & \gets \arg\min\limits_{\bar{z}} \bigg(
	g(N\bar{z}) + (\rho N/2) ||\bar{z} - \bar{x}^{k+1} - u||_2^2
\bigg) \\
u^{k+1} & \gets u^k + (\bar{x}^{k+1} - \bar{z}^{k+1})
\end{align}
$$
Pretty good with most measures, we replaced the original $z$ update scheme that required $Nn$ variables, with just an optimization over $n$ variables.