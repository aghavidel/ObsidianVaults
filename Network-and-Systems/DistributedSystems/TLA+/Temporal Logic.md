# Intro.

We describe a type of logic that considers systems of sequential steps and states.

State and its meaning are the same as the one in [[Temporal Logic Action]], in fact, they were drived from this logic in the first place.

>[!REMINDER]
>An execution of an algorithm is often thought of as a sequence of steps, each producing a new state by changing the values of one or more variables.

The main goal is:

>[!SUMMARY]
>Temporal Logic is the logic that defines appropriate semantics to reason about sequences of states.


## Temporal Formulas

A *temporal formula* is built by elementary first-order formulas with boolean operators and the unary operator $\square$ which means "always".

For example, here are a few temporal formulas. Every formula $E_i$ is just a simple first-order formula.
$$
\neg E_1 \wedge \square (\neg E_2)
$$
$$
\square(E_3 \rightarrow \square(E_4 \vee E_5) )
$$

Each of the elementary first-order formulas may be assigned a meaning using some interpreter $I$. Formally, we use the symbol $[\![F]\!]$ to denote the meaning of the formula $F$.

As a flash-back to [[first-order logic]], every boolean operator can be described in terms of the conjunction and negation operator. We have introduced the new unary $\square$ operator as well, so if we defined the semantic of all these 3 operator, then the semantic of temporal logic will be defined in terms of a natural extension over first-order semantics.

But that aside, a deeper problem is at play here.

Temporal logic needs to evaluate formulas over *states* as defined in [[Logic of Actions]], it is important that we try define how to interpret formulas over sequences of states.

As defined in [[Temporal Logic Action]], a sequence of states is simply called a *Behavior* which we can denote by $\sigma$ throughout this writing. So it can be written as a possibly infinite sequence of states:
$$
\sigma = \langle s_1, s_2, ... \rangle
$$
While a semantic of a formula like $F$ simply returns `true` or `false` by evaluating it in a single state, so a natural extension of the meaning of $F$ over a behavior $\sigma$, can be thought of as the meaning of that formula on all states in the behavior.

We denote this semantic with $\sigma[\![F]\!]$.

