# Intro
This note details the general code structure used for [[CCAC]].

## Code Structure



### Path Model

One of the core features of CCAC is the *path model*, which creates a suitable environment for specifying how CCAs work. We'll go over it here.

CCAC transforms the model in a series of discrete timesteps and so, the model can always be evaluated in finite series of steps. This also controls how long the program would calculate behaviors, allowing for it to be used easily on faster/slower environments.

The main definition is in `config.py` which defines the class `ModelConfig` and an input interface for it. As for class attributes:

| Variable Name | Description                                |
| ------------- | ------------------------------------------ |
| `N`           | number of flows (always 1 for now)         |
| `D`           | a parameter describing jitter in timesteps |
| `R`           | the value of round-trip time in timesteps  |
| `T`           | the number of timesteps that we'll process |
| `C`           | link rate                                  |
| `cca`         | name of the CCA scheme                                           |

These are core attributes of the path model. There are also 2 buffer attributes:

- `buf_min`: the maximum amount of buffered packets where packets *CANNOT* be dropped.
- `buf_max`: the amount of packets that can be buffered at any time. after this, packets *MUST* be dropped.

There are other definitions, let's leave them for now. Once this core path model is created, we can use it to define `variables` that we will then use in the full body of the code. The code for these definitions is implemented in `variables.py`. 

>[!NOTE] About The Number of Flows
>While CCAC paper only describes the single flow environment, the code itself can allow for any number by just creating separate lists for each flow. We'll discuss the single flow environment and thus remove the outer dimension of these lists.

The code defines the class `Variables`, which inherits some of [[pyz3]]. Let's see the definitions:

| Name      | Description                                                |
| --------- | ---------------------------------------------------------- |
| `A`       | Number of bytes sent until time `t`                        |
| `c`       | Congestion window at time `t`                              |
| `r`       | Pacing rate at time `t`                                    |
| `Ld`      | Losses detected by timeout or duplicate ACK until time `t` |
| `L`       | Number of bytes lost until time `t`                        |
| `timeout` | Whether or not we timeout at time `t`                      |
| `S`       | Total number of bytes served until time `t`                                                           |

There are parameters concerning the *Token Bucket Filter* (TBF) used in the model, but we'll ignore it for now.

Describing the dynamic of the environment using these variables, the service variable `S` describes how many packets we send here. With our model, the ACK for these bytes returns at time `t + c*R`. 

### Dynamics Of The Path Model

The dynamics of the path model is described in `model.py`. It describes many constrains that the variables above must adhere to, if they don't the path model may exhibit unrealistic behaviors. We'll discuss each one in it's own section.

During all of these sections, the variable `solver` is the solver instance that will accept all the constrains.

#### Initial Constrains

We start with zero service (i.e. a fresh connection), we also must dictate that some parameters start as positive values and remain positive as well. These include the congestion window and the pacing rate. Aside that, the values of the total loss and total detected loss are always nonnegative and start from 0. 

```python
s.add(c[0] > 0)
s.add(r[0] > 0)
s.add(Ld[0] >= 0)
s.add(L[0] >= 0)
s.add(S[0] == 0)
```

Monotonality ensures that these values don't go into the negatives. As for `c`, the CCA will dictate that instead, it is not the job of the path model to dictate it.

#### Monotonality Of Cumulative Variables

By definition, cumulative variables must be non-decreasing. This can be described by inequalities over all timesteps like the following:

```python
for t in range(1, T):
	s.add(A[t] >= A[t-1])
	s.add(Ld[t] >= Ld[t-1])
	s.add(L[t] >= L[t-1])
	s.add(S[t] >= S[t-1])
```

Enough? Well no!

There is still one condition left. Specifically, a condition that is required for liveness. If the CCA want's to perform it's operations, then there should always be at least *something* to send. This means that arriving packets cannot be completely lost. To ensure this, a monotonality condition is also enforced on $A(t) - L(t)$. So we also have:

```python
for t in range(1, T):
	s.add(A[t] - L[t] >= A[t-1] - L[t-1])
```

#### Network Constrains

These describe the physical characteristics of the network. These include:

- The constrain on channel capacity, specifically that $S(t) \leq C.t$ 
- The constrain on loss and service: $S(t) \leq A(t) - L(t)$
- Maximum buffer constrain: $A(t) - L(t) \leq C.t + b_{max}$
- Minimum buffer for loss constrain: 
$$
L(t) > L(t-1) \longrightarrow A(t) - L(t) \geq C.(t-1) + b_{min}
$$

Combining these gives us:

```python
for t in range(T):
	s.add(S[t] <= C * t)
	s.add(S[t] <= A[t] - L[t])
	s.add(A[t] - L[t] <= C * t + buf_max)
for t in range(1, T):
	s.add(Implies(
		L[t] > L[t - 1],
		A[t] - L[t] >= C * (t-1) + buf_min
	))
```

#### Loss Detection

All CCAs have a scheme that tunes the value of the retransmit timer such that it follows the RTT closely, this makes sure that a timeout indicates a lost packet at least most of the time. The problem is that this requires it's own separate specification though. For now, we make things easier, any moment where we have $S(t) = A(t) - L(t)$ and we know that there are inflight packets, a timeout occurs.

Also, a timeout before an RTT has even passed is kind of stupid, it implies that there is a problem with the CCA rather than an actual problem with the path (though a problem with the CCA is completely legitimate, RTOs can be tuned to follow the RTT quite closely, so let's leave out these cases).

So one way would be to implement this like the following:

```python
for t in range(T):
	if t < R:
		s.add(timeout[t] == False)
	else:
		there_are_inflight_packets = S[t - R] < A[t - 1]   # NOT A[t]!
		s.add(
			timeout[t] == And(
				there_are_inflight_packets,
				S[t - R] == A[t - R] - L[t - R]
			)
		)
	
```

So this means that timeouts correlate with the loss of an inflight packet, but to actually report them as a *detected* timeout, there should be an implication attached to timeouts:

```python
for t in range(T):
	s.add(Ld[t] <= L[t - R])
	s.add(Implies(timeout[t], Ld[t] == L[t]))
```

## CCA Implementations

Here, we provide the details of how each CCA was implemented:

### AIMD

AIMD is pretty simple compared to others. It requires one state variable aptly named `incr`, which is a list over timesteps. It takes boolean values, showing whether or not AIMD can increase the current value of the congestion window at the moment.

Note that these variables are given values *by the solver*, we only establish what type they have and what constrains they have in relation to other variables. We'll see that `incr` on it's own describes enough behaviors about AIMD to define it. The main constrain over `incr` is of course, *when* are we allowed to increase in the first place.

#### Increasing The Window

>[!REMINDER] AIMD Window Rule
>Window can increase by 1 only after successfully serving a whole window-worth of data. If at any point a loss occurs, the window will be cut in half.

There is of course, an implicit assumption that while we wait for the whole "window-worth" of data, the window itself does NOT change (once again, we stress that it is the solver that assigns values to the variables, we need to limit what it can actually consider). 

So one way of defining this informally would be the following:

>[!TLDR]
>Increase the window if and only if there exists some nonzero interval $\Delta t$ such that during this interval, $c(t)$ remains unchanged, but $S(t) - S(t-\Delta t) \geq c(t)$.

We need to be careful with how we express this though. We need to find the *minimum* interval such that this inequality holds, since generally, one can always increase the window by choosing arbitrarily large intervals. Finding this minimum requires a sweep on all timesteps and makes this expression a bit more complicated.

In general though, we can sweep from current `t`, backwards, until we notice that the window has changed, and then we record this value for the minimum interval. Once we have this, we check the condition on `S`, and if it does not hold, then we do not increase the window.

Here is how CCAC does it. A list `incr_constrain` is defined, we now add these constrains above:

```python
incr_constrain = []
for t in range(1, T):
	for delta_t in range(1, t):
		# This will be True, only for the minimum value of delta_t
		c_has_not_changed = And(
			[c[t - dt] == c[t] for dt in range(1, delta_t)],
			c[t - dt] != c[t - dt - 1]
		)
		incr_constrain.append(And(
			c_has_not_changed,
			S[t] - S[delta_t] >= c[t]
		))
```

This isn't still enough though!

There is still a few fringe cases. There is the case where we have high rates or the window is small, and we manage to send the whole window *in just a single step*. In this case, change in window does not matter, we will increase the window regardless. So we also have:

```python
for t in range(1, T):
	incr_constrain.append(S[t] - S[t-1] >= c[t])
```

There is also the case where the window has never actually changed (like the case where the connection has remained unused until now), in that case:

```python
c_never_changed = And([
	c[t - dt] == c[t] for t in range(1, t + 1)
])
incr_constrain.append(And(
	c_never_changed,
	S[t] - S[0] >= c[t]
))
```

Combining these, yields the constrains that govern how the window is increased.

#### Decreasing The Window

