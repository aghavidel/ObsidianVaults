# Temporal Logic Action

Temporal Logic Action, or TLA for short, is a logic designed to specify the behavior of a system, and indicate whether or not that behavior is compatible with a set of constraints.

It was created by Leslie Lamport, and was later extended to create the [[TLA+]] specification and the [[TLC Model Checker]].

Most of the following comes from a summary of TLA from Lamport. You can find it [Here](https://www.hpl.hp.com/techreports/Compaq-DEC/SRC-TN-1994-001.pdf)

>[!SUMMARY]
>A concurrent algorithm is usually defined in terms of a program.
>
>The *Correctness* of this program is verified in terms of that program exhibiting certain *Properties*.
>
>*TLA* is a simpler approach to this problem of specification. Both the program *and* the properties are defined in terms of simple logical formulas.
>
>So in brief, if $\mathcal{A}_{alg}$ specifies the algorithm and $\mathcal{A}_{prop}$ specifies the property, then correctness is equivalent with the following:
>$$
>		\mathcal{A}_{alg} \longrightarrow \mathcal{A}_{prop}
>$$
>
>One may also be able to define *fairness* and *liveness* similarly.


For a more rigorous discourse on this subject, here is [Lamport's original paper](https://lamport.azurewebsites.net/pubs/lamport-actions.pdf)


## Before We Start

Here is a quote from Lamport that needs to be remembered throughout all of this discussion and *every* time you open TLA+:

>[!IMPORTANT]
>How can we abandon conventional programming languages in favor of logic if the algorithm must be coded as a program to be executed?
>
>The answer is that we almost always reason about an abstract algorithm, not about a concurrent program that is actually executed. 
>
>By starting from a correct algorithm, we can avoid the timing-dependent synchronization errors that are the bane of concurrent programming. If the algorithms we reason about are not real, compilable programs, then they do not have to be written in a programming language.

Lamport argues that reasoning about *algorithms* in terms of actual programs is main reason that they become complicated. In case of concurrent programs, these implementation specifications are the main reason that they become absolute nightmares to debug and program.

He argues that this stems from a deceptively complicated language, i.e. the programming language, which has a logic of its own, not real mathematical logic.

He gives the following example:

>[!EXAMPLE]
>Assume the following line of Pascal code:
>
>```pascal
>y := x + 1
>```
>
>While us engineers find it boringly simple, this really isn't simple. Simply because Pascal like other languages, interprets equality as *assignment* which involves memory access behind the scenes.
>
>This does not obey any algebraic laws, if one was to subtract $y$ from both sides, it would yield:
>
>```pascal
>0 := x + 1 - y
>```
>
>Which is meaningless as far as Pascal is concerned.

Adding more complication from how functions work and how threading handles things behind the scenes, the efforts of the engineer become more focused on actually implementing the algorithm rather than verifying the correctness of that algorithm.

This is fertile ground for banging heads on keyboards and many sleepless nights, in which the unfortunate engineer must endure. TLA is motivated by the goal that reasoning about algorithms should be completely separated from its implementation, *at least* until its correctness is verified.

Once we have a correct algorithm, we can think about where to put the Mutex locks and other things.

Let's be clear, we have no beef against programming languages, but verifying correctness and invariants simply isn't what they are designed to do, nor do we have any motivation of making such a language, as long as we want them to have reasonably simple compilers. 


## Definitions
For a brief discourse on what TLA actually means, a few definitions are needed.

We aim to model finite, discrete and stateful systems that consist of a concurrent behavior on some level. Systems are defined by a set of *states* and some *transitions* between those states. 

While this eerily looks like a Finite State Machine, the specification of each state changes the meaning of this model.

>[!DEFINITION] **State**
> A state is a collection of assignments to a set of variables, that define a system.

The set of all states that make up a system is denoted with $\mathbf{St}$ .

Since TLA must verify the consistency of an *execution* of a system, we need to specify how that execution is modeled.

>[!DEFINITION] Behavior
>A Behavior, models a single execution of a system as a sequence of states: $\{S_i\}_{i=1}^{i=N}$ for $S_i \in \mathbf{St}$


>[!IMPORTANT]
>Systems are still for the most part, *real* constructs. 
>The abstraction that converts that system to a Behavior (which is just a mathematical model) is assumed to be known.
>
>In the real world, either such a specification must have been used at some point to create the system, or the system designer can find a suitable model for their system that translates into this behavioral model.

TLA uses essentially just [[First Order Logic]] to create formulas that express a system. The main difference however, is the formula is interpreted in a specific way that matches the requirements of the system.

If that interpretation returns *true*, then the behavior under evaluation is consistent with the system and therefore the system does not violate any rule, *for that specific execution*.


>[!DEFINITION] Satisfying a Formula
>Given a TLA formula $\Pi$ and a system $S$, we say that $S$ *satisfies* the formula $\Pi$, if and only if for every behavior of $S$, $\Pi$ remains *true*.

Another definition:

>[!DEFINITION] Step and Primed/Unprimed Variables
>A step is a pair of states $(s_o, s_n)$ which we simply may refer to as the *old* and *new* states. 
>
>Any variable of state $s_o$ is called *Unprimed*
>Any variable of state $s_n$ is called *Primed*

>[!NOTE]
>A formula that is evaluated over a single step is called an ***Action***.

For example:

>[!EXAMPLE]
>An action $\mathcal{X}$ that operates on two states, consisting of variables $x$ and $y$.
>
>![[Pasted image 20220912005925.png]]
>
>Here, the primed variables are shown with a prime. (duh!)


## TLA Formula Building Blocks

The main foundation of TLA comes from the [[Logic of Actions]], with a few extensions:

Much like first order logic, we have:
- Predicates $P_1, P_2, ...$
- Functions (of states) $f_1, f_2, f_3$
- Constants $c_1, c_2, c_3, ...$
- Variables $x, y, z, ...$
- Connectives:
	- Conjunction: $\wedge$
	- Disjunction: $\vee$ 
	- Implication: $\rightarrow$
	- Negation: $\neg$
	- Equivalence: $\equiv$, sometimes also used for *Bidirectional Implication* $\iff$
* *Temporal Symbols* (these are new)
	* Temporal Existential Quantifier: $\exists$ 
	* Prime: $\prime$ 
	* Always: $\square$












