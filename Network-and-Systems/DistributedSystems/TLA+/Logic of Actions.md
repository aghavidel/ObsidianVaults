# Intro.

We won't go too far in details.

We want to create a logic that interprets *actions* on variables as logical formulas.

## Definitions

### Values, Variables and States

>[!DEFINITION] Values
>We assume the set $\textbf{VAL}$ as the set of all values, consisting of:
>
>- Numbers (i.e. $1, 9, -15$)
>- Strings, usually treated as they are in programming languages. For example `"abc"`
>- Sets, like $\textbf{NAT}$ for natural numbers
>
>It really isn't necessary to define this set precisely. We don't include the values `true` and `false` within this set, we consider them a separate set of boolean values.

An algorithm at its core, is simply a procedure that assigns values to *variables* in a series of steps and stops at some point. So we have:

>[!DEF] Variables
>An infinite set of variables, as defined in [[first-order logic]].

Formulas are created through normal means in first-order logic, using the above definitions and other logical symbols. The main difference however, is the semantics of these formulas:

### Semantics

Judging whether or not a formula is "true", requires an interpretation of that formula, which concerns the semantics of this logic.

A semantic $[\![ F ]\!]$  assigns a meaning to each object in the formula $F$.

Semantics of this logic are defined not in terms of interpreters, but as *states*.

>[!DEF] State
>An assignment of values to variables is defined as a state. It is thus a mapping from $\textbf{VAR}$ to $\textbf{VAL}$.
>
>For example, a state $s(x)$ can assign a value to the variable $x$. Formally, we write $s(x)$ as $s[\![x]\!]$, since parenthesis will be used with functions instead.
>
>The collection of all states is denoted by $\textbf{St}$.

As in first-order logic, a distinction must be made between constants and their value after some semantic is applied. The mapping are assumed to be the natural mappings of variables to values.

For example, the constant symbol `3` when used in a formula, simply denotes a constant symbol which we borrow from first-order logics, it's meaning however (i.e. the mapping from this symbol to something in the $\textbf{VAL}$ set) is considered to be just the number $3$.


### State Functions and Predicates

This logic of course, contains a series of non-boolean expressions. These are *Predicates* and *Functions* that act on states. 

#### Functions
A state function is a non-boolean expression of constants and variables and operators.

Just like in first-order logic, we have semantics for some widely used operators which we also may write in a more simple manner. 
For example, the operator $+(., .)$ isn't written as $+(x, y)$ when used in a formula, but rather written as $x + y$. The semantics also translate it to $s[\![x+y]\!] = s[\![x]\!] + s[\![y]\!]$ which is very natural.

Same goes for things like multiplication and powers. So an example of a state function will be the following:

$$
	f = x^2 + y + \mathbf{3}
$$
An it's meaning under the state $s$ will be:
$$
s[\![f]\!] = s[\![x]\!]^2 + s[\![y]\!] + 3
$$
#### Predicates
Like functions, predicates are an expression of constants and variables, but they are paired with a semantic that maps them to the boolean values `true` and `false`.

These predicates usually correspond to boolean expression or assertions.


### Actions

An action is a boolean-valued expression formed from variables, primed variables, and constants. The primed values are associated with the value of the variable in the *next* state, so an action essentially takes a state and maps it to another state.

>[!EXAMPLE]
>Two action on variables $x, y$:
>$$
>	x^{\prime} + 1 = y
>$$
>$$
>	x - 1 \notin z^{\prime}
>$$

So, since an action involves two states, then a new symbol for the meaning of an action must be created. The meaning of an action $\mathcal{A}$ that operates on state $s$ to produce state $t$ is written as:
$$
s[\![\mathcal{A}]\!]t \triangleq \mathcal{A}[v|s[\![v]\!]][v'|t[\![v]\!]] \quad \text{for all variables $v$ in $\mathcal{A}$}
$$
So for example, the semantic of the previous action $x' + 1 = y$ would from $s$ to $t$ would be $t[\![x]\!] + 1 = s[\![y]\!]$.

With the above description of action, the pair of states $(s, t)$ is simply called an "$\mathcal{A}$ step".

>[!NOTE]
>With the above definition, a predicate can be essentially an action with the states $s$ and $t$ being the same.


### Rigid and Flexible Variables

Many algorithms are described in terms of parameters that remain constant and indeed, *cannot* be changed during every execution of that algorithm.

For example, an algorithm that colors a graph in $n$ colors, will not suddenly introduce a 4th color during an execution, if it was supposed to start with 3 colors in the first place. Considering that these parameters certainly *look* like variables when we discuss formulas, and cannot be replaced by constants, we define the notion of *Rigid* variables vs *Flexible* variables:


>[!DEF] Rigid and Flexible Variables
>Rigid variables, are variables that never change during any possible atomic action of a specification. Any other variables is considered a flexible variable (or simply, variable).




### The $Enabled$ Predicate

This is an special predicate. We say that an action $A$ is *enabled* at $s$ if and only if there is a state $t$ such that $s \rightarrow t$ is an A-step. This is expressed as $\text{Enabled} A$ .