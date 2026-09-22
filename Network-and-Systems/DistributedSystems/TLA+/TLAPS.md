# TLA$^+$ Proof System (TLAPS)

**Note:** This document is just an introduction to taking baby-steps about using TLAPS. This is a lot less about the formal definitions and a lot more just a tutorial and me crunching down the sources into something manageable.

We have used TLA$^+$ to write specifications, and we used TLC to model check them. Model checking always requires us to define the search space in some way (usually using parameters that we pass to the specification using `CONSTANT` statements), and thus it is not enough to *prove* properties about a specification.

It is invaluable to catch bugs, but we can never be bug-free just using TLC. For this reason, a specification needs to be proven correct by actually using the logical reasoning underneath it to see if the properties that we assert are true.

TLA$^+$ is no exception to this rule, and indeed one can write actual mathematical proofs similar to what we'll see when using propositional logic. 
However, when it comes to checking distributed systems, most computer scientists usually do not go for this method directly, because:

- Writing a specification is difficult, even if the underlying algorithm is well defined, since TLA$^+$ adds extra semantics that we need to push through, and also we might just make mistakes when writing the specification.
- Checking a specification is not at all as mechanical as it may seem. While a specification always ends up with some giant boolean expression, checking properties about it is not trivial. Using SMT solvers for example barely yields anything without significant hand-holding, because despite being made up of elementary expressions, they are both too big usually, and they also contain *temporal* expressions at times.

Thus, people generally prefer to prove things about their *algorithms*, not their specifications. That has been how most algorithms have been proved until now (when Paxos was created by Lamport, he had not yet came up with TLA$^+$ and yet we still understood why its safety properties hold, the case for liveness though is much more complicated ...).
However, there are real benefits in *machine-checking* these proofs, even if we are unable to generate them straight up, and that is what TLAPS is for.

I will devote the remainder of this text to describing how to use it and an example that comes straight from the [source]([TLA+ Proof System](https://proofs.tlapl.us/doc/web/content/Home.html)) (with some extra descriptions by me), along with discussions about liveness proofs, which for most intents and purposes is the hardest thing to do with this proof system.

> The proof system as a whole is on [Github]([tlaplus/tlapm: The TLA⁺ Proof Manager](https://github.com/tlaplus/tlapm/tree/main)).
> It is worth looking to the [example]([tlapm/examples at main · tlaplus/tlapm](https://github.com/tlaplus/tlapm/tree/main/examples)), and the TLA [libraries]([tlapm/library at main · tlaplus/tlapm](https://github.com/tlaplus/tlapm/tree/main/library)) that TALPS defines.
## A Simple Proof

Assuming that the toolbox is already installed, TLAPS can be installed by just downloading the JAR file from the source website. On Linux at least, it works out of the box with no fuss.
So I will not tarry on setting the thing up, and will go straight to the actual specification.

The proof that we will discuss is about Euclid's algorithm for finding the GCD of two numbers. The algorithm is very well known, basically:
- Given two natural numbers $M$ and $N$:
	- If $M = N$ then the GCD is $M$
	- If $M < N$, then $\text{GCD}(M, N) = \text{GCD}(M, N - M)$
	- If $M > N$, then $\text{GCD}(M, N) = GCD(M - N, N)$

This can be expressed with the following PlusCal code:
```
------------------------------- MODULE euclid -------------------------------
EXTENDS Integers

CONSTANTS M, N

(*
--algorithm euclid
variables 
    x = M,
    y = N
define
    LeftDividesRight(p, q) == 
	    \E k \in 1..q: q = k * p
    Maximum(nonemptySet) == 
	    CHOOSE a \in nonemptySet: \A b \in nonemptySet: a \geq b
    Divisors(n) == 
	    {a \in 1..n: LeftDividesRight(a, n)}
	JointDivisors(a, b) == 
		Divisors(a) \cap Divisors(b)
    GCD(a, b) == 
	    Maximum(JointDivisors(a, b))
end define;
process gcdCalc = 1 
begin
GCDCalc:
while x # y do
    if (x < y) then
        y := y - x;
    else
        x := x - y;
    end if;
end while;
end process;
end algorithm;
*)
=============================================================================
```
This is simple enough as-is. What we want to show however is that when $x = y$, then $x = \text{GCD}(M, N)$.

We can express this in many ways, but I am going to choose two:

- At each step, when $x = y$, then we are done and we have the correct GCD, which can be written as:
	  $TemporalProp1 \triangleq \square ((x = y) \implies (x = \text{GCD}(M, N)))$
- Or we can phrase it in a slightly more complex, but general way. We can say that eventually, the algorithm terminates with the correct GCD value, which can be expressed as:
	  $TemporalProp2 \triangleq {\huge \diamond} \square~\big((x = y) \wedge (x = \text{GCD}(M, N))\big)$ 
  That is, eventually we reach the state that $x = y$ and the value is the correct GCD, and we stay there forever.

The second version looks fussy, but it is the exact sort of properties that we often desire from distributed systems. We will prove both of these and show how the steps differ.

>[!IMPORTANT] Before You Write Any TLAPS Script
>Make sure to first **prove** your properties by hand, even informally.
>TLAPS really isn't a theorem prover, it is just a proof *system*, meaning that all that it does is to mechanically check that each step of a *given* proof makes sense mathematically speaking.
>Thus, if you don't have that proof yet, using TLAPS would only confuse you! You should at least have a sketch of the proof ready before you start fiddling with TLAPS.

### Informal Proof

The "informal" proof of Euclid's algorithm is rather simple. It starts by first noting that the algorithm must terminate, in particular it cannot possibly take any more than $MN$ steps for it to terminate since it would eventually hit $\text{GCD}(1, 1)$ in that case which is 1 (whether that is the correct result or not does not matter for this argument).

The proof that the final value is correct would rely on 3 properties of GCD:
- $\forall p \in \mathbb{N}: \text{GCD}(p, p) = p$ 
- $\forall p, q \in \mathbb{N}: \text{GCD}(p, q) = \text{GCD}(q, p)$ 
- $\forall p, q \in \mathbb{N}: (p > q) \implies \text{GCD}(p, q) = \text{GCD}(p - q, q)$
These are easy to show from the definition of GCD, we will omit them here but we will discuss them in the formal proof, and actually show how to prove them (even though this kind of falls into the habit of reinventing the wheel, which you really shouldn't do on an actual proof)

Using these, it is easy to see that our first theorem is actually an inductive invariant:
- It is true in the initial step since:
	- If $M = N$, then $\text{GCD}(M, N) = M = x$ 
	- If $M \neq N$ then then the statement is trivially true
- Now assume up to some intermediate step, the invariance has been true. Let $d = \text{GCD}(M, N)$; there are 3 cases for our current step:
	- $x < y$ : Then on the next step we would compute:
		  $\text{GCD}(x, y - x) = \text{GCD}(x, y) = d$
	- $x > y$ : Same as above
	- $x = y$ : Then the algorithm has terminated and trivially we would have $x = d$.

So now, let us formalize this.

## Proving Inductive Invariants

When we have a specification $Spec \triangleq Init \wedge \square~[Next]_{\text{vars}}$, and we want to prove something like:
$$Spec \implies \square~P$$
For some arbitrary state predicate $P$, then the formal induction procedure would be to find some invariance like $Inv$ that reduces to $P$, then prove the following in order:
$$
\begin{align*}
	\text{(1)} &\quad Init \implies Inv &&\quad \text{(Initial Step)} \\
	\text{(2)} &\quad Inv \wedge [Next]_{\text{vars}} \implies Inv' &&\quad \text{(Inductive Step)} \\
	\text{(3)} &\quad Inv \implies P &&\quad \text{(Reduction)}
\end{align*}
$$
These 3 steps together form a temporal tautology that is equivalent to:
$$
Init~{\huge\wedge}~\square[Next]_{\text{vars}} \implies \square~P
$$
Which is exactly what we wanted to prove.

>[!FAQ] A Heuristic For Choosing $Inv$
>Finding and inductive invariant that reduces to $P$ is not easy. However, we have found it fruitful to try and at least test things of the form $P \wedge G$ for some arbitrary expression $G$.
>These can also come in handy when discussing liveness (see [[#Proving `TemporalProp2`]])

To see this in action, I am going to make use of a simpler state invariant that is nonetheless, quite useful. This is the *Type Invariant* that you have probably seen in virtually every TLA$^+$ specification.

To formalize this, first note that the assumption that $M$ and $N$ are not zero permeates the whole proof, so we should note that.

This can be done by introducing an *assumption*:
```
NumberSet == Nat \ {0}
ASSUME NonzeroAssumption == /\ M \in NumberSet
                            /\ N \in NumberSet
```
>[!FAQ] `ASSUEM` vs `AXIOM`
>`ASSUME` can be used with both TLC and TLAPS. It signals to TLAPS that this is useable fact that it can invoke without proof, and it also signals to TLC to evaluate the statement.
>TLAPS reserves a keyword for this purpose, namely `AXIOM` that does the same but prevents TLC from evaluating it.

We have two variables, namely `x` and `y`, and thus the formal `TypeOK` invariant for us would be:
```
TypeOK == /\ x \in NumberSet
		  /\ y \in NumberSet
```
>[!NOTE] The `pc` Variables
>PlusCal introduces the `pc` variable to enforce labels. If you use theorems that need to contend with `pc` (which happens very often when wrestling with liveness proofs), then you also need to assert the type for `pc`. 
>`pc` is always a function from process IDs to a set of strings.
### Proving Our `TypeOK` State Invariant

>[!WARNING] How TLAPS Uses "facts"
>TLAPS breaks down a proof into small steps called *Proof Obligations*.
>Usually a proof is constructed hierarchically and in a top-down fashion. The inductive example that we described in the previous example essentially defines 3 obligations (each of the 3 steps) .
>
>The final step combines the obligations to prove our theorem (although there is a catch, we'll get to that).
>
>It is important to note that proving obligations usually mean that we  have to provide TLAPS with some starting point, a context of known information (usually some Boolean expressions that we know are true).
>
>In the above for example, `NonzeroAssumption` is one such fact that TLAPS can use anywhere. It is probably obvious that proofs may require a pretty large set of facts, and the solver backends used for TLAPS can easily get overwhelmed when the context becomes large enough, so TLAPS needs to be *explicitly told* when to invoke a fact.
>
>We will soon see how this is done.

#### Proof For Initial State

The `THEOREM` statement is used to declare obligations to TLAPS:
```
\* TypeOK holds for the initial state
THEOREM TypeOKOnInit == (Init => TypeOK)
```
A `THEOREM` statement has the following syntax:
```
THEOREM <Theorem Name> == <Expression>
PROOF
	<Proof>
```
>[!NOTE]
>Often you can omit `PROOF` and just go straight to the `<Proof>` without the keyword. I am still not sure what it does =)))

In the statement above, `TypeOKOnInit` is stated without a proof, thus it is not yet registered as an obligation. You can write this on the specification that we wrote and invoke TLAPS (on the toolbox, you would do it by pressing `ctrl + G` two times) which immediately exits and outputs something like this:
```
@!!BEGIN
@!!type:obligationsnumber
@!!count:0
@!!END
...
[INFO]: All 0 obligation proved
```
But the toolbox will still highlight the theorem, so it knows that the theorem is there, but has not yet passed it to the backends.

As just an instructive example, let us declare that this theorem is trivial (it kind of is, right?). In TLAPS, a trivial proof is signaled with `OBVIOUS`, meaning that its proof immediately follows from known facts:
```
\* Inv1 holds for the initial state
THEOREM TypeOKOnInit == (Init => Inv1)
PROOF OBVIOUS
```
Now *this* is an obligation, TLAPS will send this to the backend provers. However, attempting to prove this fails and the toolbox shows the following error:
```
(* SMT failed with status = sat *)
ASSUME NEW CONSTANT M,
       NEW CONSTANT N,
       NEW VARIABLE x,
       NEW VARIABLE y,
       NEW VARIABLE pc
PROVE  Init => TypeOK
```
Ignore the `NEW CONSTANT` and `NEW VARIABLE` statements. Essentially, the SMT backend does not think this formula can be satisfied as-is!
You can see it better if you look into the console output of the proof system. TLAPS uses multiple backends (those being at least Isabelle, Zenon and SMT), and each one will output a result to the console. The console output of TLAPS invoked by SMT should contain a line like the following:
```
...

** Unexpanded symbols: STATE_Init_, STATE_TypeOK_
...
```
Essentially, the backends don't know the meaning of the symbols `Init` and `TypeOK` (the prefix `STATE_` is added by TLAPS to say that these are state predicates), so it treats them as blank boolean variables.
So of course the theorem is wrong, just input `Init = TRUE` and `TypeOK = FALSE`.

This goes back to the warning we wrote above, TLAPS is very lazy with invoking facts, which it only does when necessary. You can see that even the *assumption* that we made about $M$ and $N$ is nowhere to be seen above. You must signal to TLAPS when each fact is needed.

The easiest way to do this is through a `BY .. DEF ..` statement. These statements have the following format:
```
BY <Assumption 1>, ... , <Assumption n> DEF <Def. 1>, ... , <Def. m>
```
In the above, assumptions are expressions taken as fact, definitions are the name of symbols that need to be expanded (i.e. replaced with what comes after `==` in their definition).
To prove our theorem, we need to expand both `Init` and `TypeOK` and also invoke `NonzeroAssumption`:
```
THEOREM CorrectOnInit == (Init => Inv1)
PROOF BY NonzeroAssumption DEF Init, TypeOK
```
In case you have a theorem that follows from assumptions, you can just drop `DEF` out completely. If you don't need assumptions, you can write `BY DEF ...` instead.

TLAPS manages to prove this. The toolbox will color the theorem green to show that it has been checked. It is worth noting that we proved this theorem *without telling TLAPS what `NumberSet` is.* TLAPS assumes that `NumberSet` is just a set, and nothing else. So this particular theorem can be satisfied without knowing the definition of that set.

If our `Init` expression for any reason contained an expression like `M + N` which does draw upon the meaning of `NumberSet` because of the `+` operator, then we would have to tell TLAPS the definition of `NumberSet` so it can reason about the expression.
#### Proof For State Transitions

We can now assume that there exists at least one state that our invariant is true, we can now argue about how it changes when we make a transition from some state. For this case, we would have to show that:
```
THEOREM TypeOKOnTransition == TypeOK /\ Next => TypeOK'
```
Since we are following this after proving that `TypeOK` is correct on the initial state, then in order to prove this theorem, it would be enough to assume `TypeOK` and `Next` are correct and show `TypeOK'` is correct.

We will however, slightly simplify this. Usually, especially for complex modules, `Next` can be very large, being the disjunction of many different actions. For our simple GCD algorithm, the PlusCal translation outputs the following:
```
vars == << x, y, pc >>

ProcSet == {1}

Init == (* Global variables *)
        /\ x = M
        /\ y = N
        /\ pc = [self \in ProcSet |-> "GCDCalc"]

GCDCalc == /\ pc[1] = "GCDCalc"
           /\ IF x # y
                 THEN /\ IF (x < y)
                            THEN /\ y' = y - x
                                 /\ x' = x
                            ELSE /\ x' = x - y
                                 /\ y' = y
                      /\ pc' = [pc EXCEPT ![1] = "GCDCalc"]
                 ELSE /\ pc' = [pc EXCEPT ![1] = "Done"]
                      /\ UNCHANGED << x, y >>

gcdCalc == GCDCalc

Terminating == /\ \A self \in ProcSet: pc[self] = "Done"
               /\ UNCHANGED vars

Next == gcdCalc
           \/ Terminating

Spec == Init /\ [][Next]_vars

Termination == <>(\A self \in ProcSet: pc[self] = "Done")
```
There is a bit of redundancy in this specification, but learning to work around that is invaluable so that we can use TLAPS with PlusCal.
To prove that state transitions are OK, we need to show that `TypeOK` remains true under every `gcdCalc` (equivalently `GCDCalc`) or `Terminating` step. We'll do this for `GCDCalc` since it is more interesting:
```
THEOREM TypeOKOnGCDCalc == (TypeOK /\ GCDCalc) => TypeOK'
```
To prove this, we would have to assume `TypeOK` and `GCDCalc` are true and show that `TypeOK'` follows as well.

>[!REMINDER] Primed Expression
>Remember that primed expressions of the form $F'$ are just the expression $F$ with all its variables replaced with primed versions.
>At the end of the day, primed variables are considered no differently compared to unprimed variables, they are just boolean values.

In TLAPS terms, we would have to make `TypeOK` and `GCDCalc` usable facts for the proof that follows. If we use an `ASSUME` statement, then we would have to remind TLAPS at each step with `BY` token which is tedious. We want to say that only in this step of the proof, we are allowed to take these expressions as facts.

So there are two things here that we have not discussed until now:
- How to *structure* a proof, such that not only does it show the step-by-step procedure of each lemma or theorem used to prove something, but also lets us control the context at each step?
- How to control things that we make known to each sub-proof, and how to make it abstract enough so that we can declare some axiom or lemma and defer its proof?

##### Proof Hierarchy

TLAPS supports hierarchical proofs in order to help cope with the first requirement above. Proofs are structured in step-by-step fashion with numbers in a list, and each list item can expand into sub steps.
- Each proof step has a *number* and optionally, a *name*.
- The format for naming steps is `<LEVEL>NAME.`. So things like `<2>a.`, `<3>12.` are valid ways to label proof steps.
- The `NAME` part may be omitted, which would constitute what follows it as an *unnamed* step.
- Each step asserts a proof *goal*, an expression that needs to be proven given the appropriate context.

>[!IMPORTANT]
>Unnamed steps are useable throughout the entire proof (i.e. all steps within a level can use it).
>Named steps must be manually added to the context using a `BY` clause, with the exception of steps that are on the same level.

A hierarchical proof would usually look like the following:
```
<1>1. [something to prove (1)]
	<2>1. [proof goal (1)]
		  [leaf proof for proof goal (1)]
	<2>2. [proof goal (2)]
		  [leaf proof for proof goal (2)]
	...
	<2>n. QED
	      [leaf proof for QED]
<1>2. [something else to prove (2)]
	...
```
Steps `<2>1` and `<2>2` all the way to `<2>n-1` are just asserting their own separate goals which is different from the goal on `<1>1`. Step `<m>j+1` in some level is allowed to add steps `<m>1 ... <m>j` to its context to prove its goal (for example, using a `BY` clause).

The final leaf (step) of a proof (sub-proof) needs to assert that it is proving the goal, so the last step of level `<2>` needs to say that "This is the last step, and what I want to prove as stated in `<1>1.` follows from all that we have put into context under `<2>` and this and that ...".
TLAPS uses the `QED` keyword for this, it essentially asserts that the goal is provable by the current context and asks TLAPS to tell the backends to verify it. Essentially, QED is a shortcut for referring to the top level proof (i.e. the first step that precedes the QED leaf and is on a lower level).
##### `SUFFICES` Construct

In service of the second requirement that we mentioned, TLAPS includes the `SUFFICES` step. The proof that we wrote for the initial step is an ordinary proof of the form `ASSUME <things> PROVE <obligations>`. 
The sub-proofs use `<things>` to prove `<obligations>` and satisfy the `PROVE` clause, and furthermore, the rest of the proof that is unrelated to `<obligations>` but is in the same level of this proof can use `<things>` to prove other things.

So as an example, take the following:
```
<4>1. t1 
	<5>1. s1 
	<5>2. s2 
	… 
	<5>m. QED 
<4>2. t2 
<4>3. t3 
… 
<4>n. QED
```
Sub-steps on level 5, are proving the assertion of `t1` (which could be an `ASSUME` statement) and the steps on level 4 after `<4>1.` can use `t1` as a fact.
Sometimes though, we may want to reverse this mechanism. We may want to first assert that we assume `t1` is true, we prove the contents of level `<4>` (which may often be easier and shorter) and then finally we justify why `t1` was usable.

This is where the `SUFFICES` construct comes in. It has the format:
```
<n>1. SUFFICES [expression1]
      PROVE [obligation1]
	<n+1>1. [step for proving obligation1 using expression1]
	...
<n>2. [step for justifying why expression1 was true]
...
```
If we take the example with `t1` that we mentioned above, `SUFFICES` allows us to write it in reverse:
```
<4>1. SUFFICES t1
	<5>1. t2
	<5>2. t3
	...
	<5>n-1. QED
<4>2. s1
<4>3. s2
...
<4>m+1. QED
```
Depending on which sub-proof is easier and shorter, this may be worth doing.

>[!FAQ] In Other Words
>`SUFFICES` asserts the top level proof can be done by assuming a bunch of things and proving something else.
>The proof that we write directly after it shows that this assertion is sound, and the proofs that follow it are allowed to invoke the assumptions we made and use the new goal that presented.

In the particular case of our proof for state transitions, this is actually helpful. We can do something like the following:
```
THEOREM TypeOKOnGCDCalc == (TypeOK /\ GCDCalc) => TypeOK'
<1> SUFFICES ASSUME TypeOK, GCDCalc PROVE TypeOK'
<1> QED
```
This means:
- We assert that in order to prove `TypeOK /\ GCDCalc => TypeOK'`, it is sufficient to assume `TypeOK` and `GCDCalc` are true and just prove `TypeOK'`. This changes the top level proof goal of `TypeOK /\ GCDCalc => TypeOK'` to `TypeOK'`.
- `QED` now refers to the *new* proof goal (i.e. `TypeOK'`)
- We now must justify both the new `QED` goal, and the assertion that the first step is indeed sufficient (in this case, this is obvious since that is the literal definition of implication in predicate logic)

To proceed with the proof, it will be helpful to make the definitions of `TypeOK` and `Next` known to the backends. We could do this with a `BY DEF` clause like we did before, but this becomes cumbersome since we have to do it for many steps, so we can make them known throughout the proof for this theorem (in particular, the current level) using another construct, `USE` :
```
THEOREM TypeOKOnGCDCalc == (TypeOK /\ GCDCalc) => TypeOK'
<1> SUFFICES ASSUME TypeOK, GCDCalc PROVE TypeOK'
<1> USE DEF TypeOK, GCDCalc
<1> QED
```
The general syntax is:
```
USE <e1>, <e2> ... DEF <d1>, <d2>, ...
```
Where:
- `<e1>` and folks are expressions that are made into a proof obligation and then added to the context (so each expression is checked to be true given the current context).
- `<d1>` and folks are definitions do be expanded throughout the proof.

>[!NOTE] `HIDE`
>`USE` will expand the definitions throughout the current level, *and* its sub-levels.
>The second one might not be desirable and make the backends slow. You can thus selectively revoke this by using `HIDE <def>` in appropriate steps.

Now for the next step, we need to consider the definition of `GCDCalc`. It is fruitful to look at the formula of each action and decide based on what the specification actually says rather than trying to force our informal proof onto the specification. We see that `GCDCalc` has an `IF` statement based on whether or not $x \neq y$ (this is inherited from the `while` loop in PlusCal). Naturally, to reason about changes in `TypeOK`, it would be reasonable to split the proof into two cases.

The expression `(x # y) \/ ~(x # y)` is a tautology, thus it is a good candidate to take each statement as fact and then prove the theorem assuming that. This is where TLAPS introduces the `CASE` construct:
```
THEOREM TypeOKOnGCDCalc == (TypeOK /\ GCDCalc) => TypeOK'
<1> SUFFICES ASSUME TypeOK, GCDCalc PROVE TypeOK'
<1> USE DEF TypeOK, GCDCalc
<1>1 \/ (x # y)
     \/ ~(x # y)
<1>a CASE (x # y)
<1>b CASE ~(x # y)
<1>2 QED
```
>[!FAQ] What Does `CASE` Do Exactly?
>`CASE` inherits the top level proof goal (in this case, `TypeOK'`) and asserts its argument as a useable fact (so anything under `<1>a` can use `BY <1>a` to refer to the fact that `x # y`).

This still gives no obligations, but assuming we prove `TypeOK'` for each case, then the QED step should follow immediately. So we can tell this to TLAPS using a `BY`clause:
```
THEOREM TypeOKOnGCDCalc == (TypeOK /\ GCDCalc) => TypeOK'
<1> SUFFICES ASSUME TypeOK, GCDCalc PROVE TypeOK'
<1> USE DEF TypeOK, GCDCalc
<1>1 \/ (x # y)
     \/ ~(x # y)
<1>a CASE (x # y)
<1>b CASE ~(x # y)
<1>2 QED BY <1>1, <1>a, <1>b
```
If we pass this to TLAPS, then it actually accepts the proof of the QED step, meaning that it does accept that if `<1>1` is always true, and we prove `TypeOK'` for cases `<1>a` and `<1>b`, then `TypeOK'` is true in general given the current assumptions.

Note that *none* of these 3 assumptions in front of the `BY` token have been justified, in fact if we drop one of the cases (like say, $x \neq y$) from `<1>1` and remove `<1>a`, then TLAPS still says that yes, the theorem is true.

So we now need to see what obligations we have left:
- Show the `SUFFICES` step is sound
- Show the cases outlined in `<1>1` are universal
- Prove the theorem assuming the cases in `<1>a` or `<1>b` can be taken as fact

The first obligation is obvious and TLAPS accepts `PROOF OBVIOUS` for it. The same applies to our tautology:
```
THEOREM TypeOKOnGCDCalc == (TypeOK /\ GCDCalc) => TypeOK'
<1> SUFFICES ASSUME TypeOK, GCDCalc PROVE TypeOK'
    PROOF OBVIOUS
<1> USE DEF TypeOK, GCDCalc
<1>1 \/ (x # y)
     \/ ~(x # y)
    PROOF OBVIOUS
<1>a CASE (x # y)
<1>b CASE ~(x # y)
<1>2 QED BY <1>1, <1>a, <1>b
```
So all we need is to prove `TypeOK'` for `<1>a` and `<1>b`. The case for `<1>b` is also obvious since that case would leave $x$ and $y$ unchanged. For `<1>a`, we need to again reason by cases, in particular whether we have $x < y$:
```
THEOREM TypeOKOnGCDCalc == (TypeOK /\ GCDCalc) => TypeOK'
<1> SUFFICES ASSUME TypeOK, GCDCalc PROVE TypeOK'
    PROOF OBVIOUS
<1> USE DEF TypeOK, GCDCalc
<1>1 \/ (x # y)
     \/ ~(x # y)
    PROOF OBVIOUS
<1>a CASE (x # y)
    <2>1 \/ (x < y)
         \/ ~(x < y)
        PROOF OBVIOUS
    <2>a CASE (x < y)
    <2>b CASE ~(x < y)
    <2>2 QED BY <2>1, <2>a, <2>b
<1>b CASE ~(x # y)
    <2> QED BY <1>b
<1>2 QED BY <1>1, <1>a, <1>b
```
In each of the cases `<2>a` and `<2>b`, $x$ and $y$ remain non-negative because of the definition of `NumberSet` (i.e. set of non-negative numbers):
```
THEOREM TypeOKOnGCDCalc == (TypeOK /\ GCDCalc) => TypeOK'
<1> SUFFICES ASSUME TypeOK, GCDCalc PROVE TypeOK'
    PROOF OBVIOUS
<1> USE DEF TypeOK, GCDCalc
<1>1 \/ (x # y)
     \/ ~(x # y)
    PROOF OBVIOUS
<1>a CASE (x # y)
    <2>1 \/ (x < y)
         \/ ~(x < y)
        PROOF OBVIOUS
    <2>a CASE (x < y)
        <3>1 (y - x) \in NumberSet
            BY <2>a DEF NumberSet
        <3>2 QED BY <1>a, <2>a, <3>1
    <2>b CASE ~(x < y)
        <3>1 (x - y) \in NumberSet
            BY <1>a, <2>b DEF NumberSet
        <3>2 QED BY <1>a, <2>b, <3>1 
    <2>2 QED BY <2>1, <2>a, <2>b
<1>b CASE ~(x # y)
    <2> QED BY <1>b
<1>2 QED BY <1>1, <1>a, <1>b
```
Note in particular the proofs of `<3>2`, which cite multiple case contexts. This concludes the proof for `GCDCalc` action. The proof for `Terminating` action is obvious as well, assuming we make `vars` known to the backends:
```
THEOREM TypeOKOnTerminating == (TypeOK /\ Terminating) => TypeOK'
<1> SUFFICES ASSUME TypeOK, Terminating PROVE TypeOK'
    PROOF OBVIOUS
<1> QED BY DEF TypeOK, Terminating, vars
```
And finally, we can conclude the proof with the following two theorems:
```
THEOREM TypeOKOngcdCalc == (TypeOK /\ gcdCalc) => TypeOK'
PROOF BY TypeOKOnGCDCalc DEF gcdCalc

THEOREM TypeOKOnNext == (TypeOK /\ Next) => TypeOK'
PROOF BY TypeOKOngcdCalc, TypeOKOnTerminating DEF Next
```
#### Proof Conclusion

We now need only stitch together our results and state the proof. For this, we might just write:
```
THEOREM TypeOKInv == (Spec => []TypeOK)
BY TypeOKOnInit, TypeOKOnNext DEF Spec
```
However, this fails; partly because of a technicality, and partly because we are not really done yet.
Let us focus on the latter, what have we missed?
Note that the definition of `Spec` was:
```
Spec == Init /\ [][Next]_vars
```
The main key thing to note here is that it is written as `[Next]_vars`, which means that our specification allows stuttering steps on `vars` (stuttering steps are a technicality that makes refinement easier). What we have missed is that we have not proven that `TypeOK` remains true under stuttering.
That part is obvious enough, we need only tell TLAPS what `vars` is:
```
THEOREM TypeOKInv == (Spec => []TypeOK)
<1>1 TypeOK /\ UNCHANGED vars => TypeOK'
     PROOF BY DEF TypeOK, vars
<1> QED BY TypeOKOnInit, TypeOKOnNext, <1>1 DEF Spec
```
However, this still does not work; the reason for this is a technicality.
The backends that TLAPS supports by default, do not understand temporal logic (i.e. SMT, Zenon and Isabelle). So they do not understand what `[]TypeOK` or even `[][Next]_vars` mean.
Fortunately, TLAPS comes with another backend, LS4 that does understand such formulas, however it is excluded by default since it is ill-suited for proving most theorems that do not contain temporal operators.
As such, it must be invoked manually. The way that TLAPS handles this is by defining certain *Pragma* tokens that the user may invoke as an assumption (perhaps as argument to the `BY` clause) to signal to TLAPS what sort of tactics it must use.

These Pragmas are defined in the `TLAPS.tla` module, which is now a standard module and we may simply extend it when we need it. The Pragma that we want to invoke LS4 for this particular proof is `PTL`:
```
THEOREM TypeOKInv == (Spec => []TypeOK)
<1>1 TypeOK /\ UNCHANGED vars => TypeOK'
     PROOF BY DEF TypeOK, vars
<1> QED BY PTL, TypeOKOnInit, TypeOKOnNext, <1>1 DEF Spec
```
This finally checks as OK, we just proved our temporal property :-).

>[!IMPORTANT] Other Backend Pragmas
>The `TLAPS.tla` module is worth a look. One thing worth noting is that it contains Pragmas `Zenon`, `SMT` and `Isa` for the backends that we discussed, as well as some other backends.
>You may invoke these under an obligation to signal to TLAPS which backend must be preferred. By default, TLAPS tries SMT first, then Zenon and finally Isabelle.
>
>Another thing worth noting is that these Pragmas can accept an integer argument as well for the timeout of the backend. For example `Isa(60)` invokes Isabelle with a 60 second timeout.
### Proving `TemporalProp1`

We want to prove $TemporalProp1$, and for that we need an inductive invariant that reduces to it. The invariant that we use is that the GCD value remains the same at each state, so essentially:
```
\* The first inductive invariant
\* GCD(M, N) is the same as GCD(x, y) at each step
Inv1 == GCD(x, y) = GCD(M, N)
```
Now, we want to show the first obligation of our inductive proof, namely that `Inv1` holds for the initial state. 
```
THEOREM CorrectOnInit == (Init => Inv1)
PROOF BY DEF Init, Inv1
```
Note that **there are still unexpanded symbols** in the above (in particular, the provers still have no idea what `GCD` even is in the expanded definition of `Inv1`), but they *don't need to know!* (for this simple theorem of course).

The main part of TLAPS that is more art than logic is to be savvy with how much we expand statements and how much context we give to the backends, because as we mentioned, they can get overwhelmed very easily.

To prove `Inv1` in general, we need some lemmas about GCD, in particular we need:
```
LEMMA GCDProp1 == \A p \in NumberSet: GCD(p, p) = p
LEMMA GCDProp2 == \A p, q \in NumberSet: GCD(p, q) = GCD(q, p)
LEMMA GCDProp3 == \A p, q \in NumberSet: (p > q) => GCD(p, q) = GCD(p - q, q)
```
>[!FAQ] What's a `LEMMA`?
>It's just a `THEOREM` with a different name. TLAPS treats it exactly the same. The distinction is merely stylistic, to signal to the reader that this is not a main result, rather it's an auxiliary one (i.e. exactly what a Lemma is in normal proofs ...).

We can state them as `AXIOMS` rather than a `LEMMA` if we don't feel like bothering to prove these, but since this is a tutorial, it may be instructive to actually prove them. Properties 1 and 2 are easy enough to show by just expanding `GCD`:
```
LEMMA GCDProp1 == \A p \in NumberSet: GCD(p, p) = p
BY DEF NumberSet, GCD, Maximum, Divisors, LeftDividesRight

LEMMA GCDProp2 == \A p, q \in NumberSet: GCD(p, q) = GCD(q, p)
BY DEF GCD
```
For property 3, we may need a bit more work.

##### Proving Property 3

TLAPS benefits from hand-holding, especially for arithmetic/set operations. Here are some trivial lemmas that we will need:
```
LEMMA FactorizationLemma1 == 
	\A a, b, c \in NumberSet: a * c - b * c = (a - b) * c
	PROOF BY DEF NumberSet
LEMMA FactorizationLemma2 == 
	\A a, b, c \in NumberSet: a * c + b * c = (a + b) * c
	PROOF BY DEF NumberSet
LEMMA ArithmeticLemma1 == 
	\A a, b \in NumberSet: (a - b) \in NumberSet <=> a > b
	PROOF BY DEF NumberSet
LEMMA ArithmeticLemma2 == 
	\A a, b, c \in NumberSet: 
		(a - b) \in NumberSet <=> (a - b) * c \in NumberSet
	PROOF BY DEF NumberSet
LEMMA ArithmeticLemma3 == 
	\A a, b, c \in NumberSet: 
		a * b = c => (a \leq c) /\ (b \leq c)
	PROOF BY DEF NumberSet
```
With these, we can prove the property by showing that:
```
LEMMA JDLemma == 
	\A p, q \in NumberSet: 
		(p > q) => (JointDivisors(p, q) = JointDivisors(p - q, q))

LEMMA GCDProp3 == \A p, q \in NumberSet: (p > q) => GCD(p, q) = GCD(p - q, q)
PROOF BY JDLemma DEF GCD, NumberSet
```
`JDLemma` holds, because if $d~|~p$ and $d~|~q$, then there must be $m, n \in \mathbb{N}$ such that $md = p$ and $nd = q$. If we have $p > q$, then $p - q = (m - n)d > 0$ and thus $m - n > 0$, which directly says that $d~|~p-q$. The inverse holds as well, if $d~|~p-q$ and $d~|~q$, then $d~|~p$ as well.

In terms of what we have written in the specification, this can be used to prove the lemma. In particular, we can prove it like this:
```
LEMMA DivLemma1 == 
	\A p, q \in NumberSet: 
		(p > q) => \A d \in JointDivisors(p, q): 
			d \in JointDivisors(p - q, q)
LEMMA DivLemma2 == 
	\A p, q \in NumberSet: 
		(p > q) => \A d \in JointDivisors(p - q, q): 
			d \in JointDivisors(p, q)

LEMMA JDLemma == 
	\A p, q \in NumberSet: 
		(p > q) => (JointDivisors(p, q) = JointDivisors(p - q, q))
PROOF BY DivLemma1, DivLemma2
```
TLAPS accepts that `DivLemma1` and `DivLemma2` would prove `JDLemma` and by extension `GCDProp3`, so we need only prove `DivLemma1` and `DivLemma2`. For these, we need to essentially do the same informal proof that we mentioned above, by showing the properties we listed for $m,n,p$ and $q$. We need to supply these definitions and make them known throughout the proof.

We can do with the `NEW` syntax like the following:
```
LEMMA DivLemma1 == 
	\A p, q \in NumberSet: 
		(p > q) => \A d \in JointDivisors(p, q): 
			d \in JointDivisors(p - q, q)
<1>1 ASSUME
    NEW p \in NumberSet,
    NEW q \in NumberSet,
    p > q,
    NEW d \in JointDivisors(p, q)
    PROVE d \in JointDivisors(p - q, q)
<1>2 QED BY <1>1
```
Essentially, we say that we pick $p$ and $q$ from the set of natural numbers such that we can assume $p > q$ and then we pick a divisor, $d$ from the set of joint divisors of $p$ and $q$ and then we say that it must also be in the set of joint divisors for $p-q$ and $q$.
TLAPS accepts the proof for `<1>2` but the proof is still unfinished.

Despite the simplicity of this, TLAPS needs (as it often does) quite a bit of handholding with arithmetic operations, thus the proof for `<1>1` is actually much longer than one would expect. We supply one proof, which also makes use of another syntax that we have not mentioned, the `PICK` syntax:
```
NEW p \in NumberSet,
NEW q \in NumberSet,
p > q,
NEW d \in JointDivisors(p, q)
PROVE d \in JointDivisors(p - q, q)
<2> d \in NumberSet
	PROOF BY DEF JointDivisors, NumberSet, Divisors
<2>1 PICK m \in NumberSet: m * d = p
	<3>1 d \in Divisors(p)
		BY <1>1 DEF JointDivisors
	<3>2 \E k \in NumberSet: k * d = p
		BY <3>1 DEF NumberSet, Divisors, LeftDividesRight
	<3> QED BY <3>2
<2>2 PICK n \in NumberSet: n * d = q
	<3>1 d \in Divisors(q)
		BY <1>1 DEF JointDivisors
	<3>2 \E k \in NumberSet: k * d = q
		BY <3>1 DEF NumberSet, Divisors, LeftDividesRight
	<3> QED BY <3>2
<2>3 m * d - n * d = p - q
	PROOF BY <2>1, <2>2, <1>1 DEF NumberSet
<2>4 m * d - n * d = (m - n) * d
	PROOF BY <2>3, FactorizationLemma1
<2>5 (m - n) * d = p - q
	PROOF BY <2>4, <2>3
<2>6 p - q \in NumberSet
	PROOF BY <1>1, ArithmeticLemma1
<2>7 m - n \in NumberSet
	PROOF BY <2>5, <2>6, ArithmeticLemma2
<2>8 m - n \leq p - q
	PROOF BY <2>5, <2>6, <2>7, ArithmeticLemma3
<2>9 m - n \geq 1
	PROOF BY <2>7 DEF NumberSet
<2>10 m - n \in 1 .. (p-q)
	PROOF BY <2>8, <2>9 DEF NumberSet
<2>11 \E k \in  1..(p-q): p - q = k * d
	PROOF BY <2>10, <2>5
<2>12 LeftDividesRight(d, p - q)
	BY <2>11 DEF LeftDividesRight
<2>13 d \leq p - q
	BY <2>5, <2>6, <2>7, ArithmeticLemma3
<2>14 d \in 1 .. (p-q)
	PROOF BY <2>13 DEF NumberSet
<2>15 d \in Divisors(p - q)
	BY <2>12, <2>14 DEF Divisors
<2>16 d \in JointDivisors(p - q, q)
	BY <2>15, <1>1 DEF JointDivisors
<2> QED BY <2>16
```
The `PICK` syntax looks like:
```
PICK <things>: <conditions>
	<Proof of why a thing even exists to do this>
```
Thus, on `<2>1` and `<2>2`, we make `m` and `n` known in numbered steps that we can use later, and in the sub-proofs (the ones on level 3), we justify why such numbers even existed.
The rest of the proof should be self-explanatory, if rather strangely wordy.
The proof for `DivLemma2` is largely the same, so we omit it here.

##### Finishing The Proof
Now that we have `GCDProp3`, we can prove `TemporalProp1`. To do this, we again come up with another invariant that reduces to `TemporalProp1`, in particular, we can show:
```
Inv1 == GCD(x, y) = GCD(M, N)
```
This means that at each step, the `GCD` result remains unchanged, until `x = y` where by `GCDProp1` we would have calculated the result immediately. TLAPS accepts that `Inv1` holds in the initial state immediately:
```
THEOREM CorrectOnInit == (Init => Inv1)
PROOF BY DEF Init, Inv1
```
Now for state transitions, we are actually going to invoke `TypeOK` as well. The reason that we do this is that in order to use `GCDProp3` for the case where `x # y` and `~(x < y)`, then we need to deduce that `x > y`. This is obvious only if we *know* that `x` and `y` are always numbers. TLAPS does not know this immediately, but we have already proved it when we showed that `TypeOK` is an invariant.

Thus, we are going to use this to helps us, we prove the following instead:
```
THEOREM Inv1OnTerminating == TypeOK /\ Inv1 /\ Terminating => Inv1'
PROOF BY DEF Inv1, TypeOK, Terminating, vars

THEOREM Inv1OnGCDCalc == TypeOK /\ Inv1 /\ GCDCalc => Inv1'

THEOREM Inv1OnNext == (TypeOK /\ Inv1 /\ Next) => Inv1'
PROOF BY  Inv1OnGCDCalc, Inv1OnTerminating DEF gcdCalc, Next
```
And then finally we would have:
```
THEOREM Inv1Inv == (Spec => []Inv1)
<1>1 Inv1 /\ UNCHANGED vars => Inv1'
    BY DEF Inv1, vars
<1>2 QED BY <1>1, Inv1OnInit, Inv1OnNext, TypeOKInv, PTL DEF Spec
```
In the final step, we are using the fact that if we have $Spec \implies \square~(TypeOK \wedge Inv1)$ *AND* we know that $Spec \implies \square~TypeOK$, then we can deduce that $Spec \implies \square~Inv1$. So we now only need to prove `Inv1OnGCDCalc`. A proof can be done as follows:
```
THEOREM Inv1OnGCDCalc == TypeOK /\ Inv1 /\ GCDCalc => Inv1'
<1> SUFFICES ASSUME TypeOK, Inv1, GCDCalc PROVE Inv1'
    PROOF OBVIOUS
<1> USE DEF Inv1, TypeOK
<1>1 \/ (x # y)
     \/ ~(x # y)
    PROOF OBVIOUS
<1>a CASE (x # y)
    <2>1 \/ (x < y) 
         \/ ~(x < y)
        PROOF OBVIOUS
    <2>a CASE (x < y)
        <3>1 y - x \in NumberSet
            BY <2>a DEF NumberSet
        <3>2 /\ y' = y - x
             /\ x' = x
            BY <1>a, <2>a, <3>1 DEF GCDCalc
        <3>3 GCD(x', y') = GCD(x, y - x)
            BY <3>2
        <3>4 GCD(x', y') = GCD(y - x, x)
            BY <3>3, GCDProp2, <3>1
        <3>5 GCD(y - x, x) = GCD(y, x)
            BY GCDProp3, <3>1, <2>a
        <3>6 GCD(x', y') = GCD(y, x)
            BY <3>4, <3>5
        <3>7 GCD(x', y') = GCD(x, y)
            BY <3>6, GCDProp2
        <3> QED BY <3>7
    <2>b CASE ~(x < y)
        <3>1 (x - y) \in NumberSet BY <1>a, <2>b DEF NumberSet
        <3>2 /\ y' = y
             /\ x' = x - y
            BY <1>a, <2>b, <3>1 DEF GCDCalc
        <3>3 GCD(x', y') = GCD(x - y, y)
            BY <3>2
        <3>4 (x > y)
            BY <1>a, <2>b DEF NumberSet
        <3>5 GCD(x, y) = GCD(x - y, y)
            BY GCDProp3, <3>1, <3>4
        <3>6 GCD(x', y') = GCD(x, y)
            BY <3>3, <3>5
        <3> QED BY <3>6
    <2>2 QED BY <2>1, <2>a, <2>b
<1>b CASE ~(x # y)
    <2> QED BY <1>b DEF vars, GCDCalc
<1> QED BY <1>1, <1>a, <1>b
```
So we proved `Inv1`, now for `TemporalProp1`:
```
TemporalProp1 == ((x = y) => (x = GCD(M, N)))

THEOREM TemporalProp1_Proof == Spec => []TemporalProp1
<1>1 TypeOK /\ Inv1 => TemporalProp1
    BY GCDProp1 DEF Inv1, TypeOK, TemporalProp1
<1>2 Inv1 /\ UNCHANGED vars => Inv1'
    BY DEF Inv1, vars
<1> QED BY PTL, <1>1, TypeOKInv, Inv1OnInit, Inv1OnNext, <1>2 DEF Spec
```
## Proving `TemporalProp2`

This is where TLAPS has shortcomings. Proving temporal properties of the form ${\huge\diamond}P$ or ${\huge\diamond}~\square~P$ has no predefined recipe like what we had for $\square~P$.  For these, we have to rely on auxiliary variables, other invariants and lemmas or history variables.

For `TemporalProp2`, we were asserting that:
$$
\text{TemporalProp2} \triangleq {\huge\diamond}\square\big( (x = y) \wedge (x = GCD(M, N)) \big)
$$
This property might appear weaker than what we have, however, it asserts that not only do we always terminate by reaching $x = y$, it also asserts that we eventually hit the correct value of GCD at some point and keep it.
For Euclid's algorithm, such a property expression seems needlessly fussy, but for general distributed systems, this is quite an important type of expressions for us to learn how to prove (in fact, this form is the de-facto liveness property that systems want to specify). This type of property is popular because:
- It is easy to understand. It abstracts away the *transient* behavior of a system when subject to some sequence of events.
- It deals directly with the *steady* state of a system, giving us the justification that our system not only *will* reach a good state, but it will also never get stuck in a bad state forever.
### Proving Liveness Properties

Proving liveness is always much more difficult. Really the only general way to establish liveness is to come up with really well-made invariants and use temporal tautologies (or other things that we can quickly show) to prove liveness.
This example is no different.

First, a bit of intuition will help. Our `TemporalProp2` states that we:
1. Eventually hit a state where $x = y$ and $x = GCD(M, N)$ (not necessarily once!).
2. When this happens, we will stay there forever (although we may have to reach that state multiple times for this to happen ...).

The above is actually sufficient for us to make a recipe for proving eventually-always properties as well (although it may not be the easiest one); If we have a specification $Spec \triangleq Init \wedge \square~[Next]_{\text{vars}}$, and we want to prove something like:
$$Spec \implies {\huge\diamond} \square~P$$
For some arbitrary state predicate $P$, then the formal induction procedure would be to find some expression like $G$ that reduces to $P$, then prove the following in order:
$$
\begin{align*}
	\text{(1)} &\quad G \wedge [Next]_{\text{vars}} \implies G' &&\quad \text{($G$ is self-sustaining)} \\
	\text{(2)} &\quad Spec \implies {\huge\diamond}G &&\quad \text{($G$ eventually holds)} \\
	\text{(3)} &\quad G \implies P &&\quad \text{($G$ implies $P$)}
\end{align*}
$$
We are quite familiar with the first and last steps, but the second step is a bit more elusive. At its core, it is demanding for us to pinpoint *where* $P$ becomes true and why it can be reached. We know where $P$ becomes true in our case with Euclid's algorithm, but the argument for determining why we gravitate to that from any initial state is something we have not done yet.

Indeed, it is step 2 that proves the most challenging many times, and at least to this author at this time, there is no recipe for proving it. One thing that helps the most though is to use helper invariants to make things easier. In this case, we already have one such invariant in form of `TemporalProp1`.
Since $Spec \implies \square ((x = y) \implies (x = \text{GCD}(M, N)))$ then we can simplify step 3 above and show only that $Spec \implies {\huge\diamond}(x = y)$ , since if this is true, then `TemporalProp1` gives us $x = \text{GCD}(M, N)$ and we get step 3 verbatim.

So the proof sketch is the following:
```
TemporalProp2 == ((x = y) /\ (x = GCD(M, N)))

THEOREM Liveness == Spec => <>[]TemporalProp2
<1>1 TemporalProp2 /\ Next /\ TypeOK => TemporalProp2'
    <2> SUFFICES ASSUME TemporalProp2, Next, TypeOK PROVE TemporalProp2'
    <2> USE DEF TemporalProp2
    <2>a CASE GCDCalc
        <3>1 ~(x # y)
            BY DEF TypeOK
        <3> QED BY <2>a, <3>1 DEF GCDCalc
    <2>b CASE Terminating
        BY <2>b DEF vars, Terminating
    <2> QED BY <2>a, <2>b DEF Next, gcdCalc
<1>2 TemporalProp2 /\ UNCHANGED vars => TemporalProp2'
    BY DEF vars, TemporalProp2
<1>3 Spec => <>TemporalProp2
    <2>1 Spec => <>(x = y)
	    PROOF OMITTED
    <2> QED 
        BY PTL, <2>1, TemporalProp1_Proof DEF TemporalProp1, TemporalProp2
<1> QED BY PTL, <1>1, <1>2, <1>3, TypeOKInv DEF Spec, TemporalProp2
```
Neat! Now all we need to do is show step `<2>1`, that `Spec => <>(x = y)`. As we described, this is where things get tricky. Informally, this holds since each step reduces $x$ or $y$ if they are not equal, so we either hit 1 on both or we terminate with $x = y$, which holds even if both variables are 1.
So, here is our proposed way to prove this. Define:
$$
\forall k \in \text{Nat}: P(k) \triangleq {\huge\diamond}\big((x = y) \vee (x+y \leq k)\big)
$$
This obviously holds for $k = M + N$ due to the initial state. Every non-stuttering step that the algorithm takes also reduces $x+y$ by $\min(x, y)$, which is at least 1, so if $P(k)$ is true and $x \neq y$ is true, then we can take a non-stuttering step to show that $P(k-1)$ must also be true.
We can do a downward induction to get $P(0)$, which obviously cannot let $x + y \leq 0$ since the variables are natural numbers, so we get $x = y$ instead and we are done.

This proof only holds if it is possible to take a non-stuttering step when we need it. Essentially, we need a way to formally state that "it is valid to assume that a particular step that is enabled, can be taken as fact, to prove another thing".

This brings us neatly, to the painful discussion of **fairness** ...

#### An Example

Here, we will discuss a very good [example]([[tlapm/examples/SimpleEventually.tla at main · tlaplus/tlapm](https://github.com/tlaplus/tlapm/blob/main/examples/SimpleEventually.tla)]) of a liveness proof. I'll use an older version of it that is even simpler, since I think that will help more. This one is straight TLA$^+$ for simplicity:
```
EXTENDS TLAPS

VARIABLE x, y
vars == <<x, y>>

Init == /\ x = FALSE
	    /\ y = FALSE

A == /\ x = FALSE
     /\ x' = TRUE
     /\ UNCHANGED y

B == /\ y = FALSE
     /\ y' = TRUE
     /\ UNCHANGED x

Next == A \/ B

Spec == Init /\ [][Next]_vars

TypeOK == /\ x \in BOOLEAN
          /\ y \in BOOLEAN

Prop == <>(x = TRUE)
```
This is simple, we have two steps, one changes `x` to `TRUE` if it is `FALSE`, one changes `y` to `TRUE` if it is `FALSE`. We want to show that eventually `x = TRUE` holds.

Let us write the proof for `TypeOK` being an invariant:
```
THEOREM TypeOK_inv == Spec => []TypeOK
<1> USE DEF TypeOK
<1>1 Init => TypeOK
    BY DEF Init
<1>2 TypeOK /\ [Next]_vars => TypeOK'
    <2>1 TypeOK /\ A => TypeOK'
        BY DEF A
    <2>2 TypeOK /\ B => TypeOK'
        BY DEF B
    <2>3 TypeOK /\ UNCHANGED vars => TypeOK'
        BY DEF vars
    <2> QED BY <2>1, <2>2, <2>3 DEF Next
<1> QED BY PTL, <1>1, <1>2 DEF Spec
```
Simple enough, now for the main part ...
We need a proof like this:
```
THEOREM Spec => Prop
<1> SUFFICES ASSUME []TypeOK PROVE Spec => Prop
    BY TypeOK_inv
<1> QED
```
Proving things that are wrong is impossible without making a mistake, so before we head to prove this, we'll need to check this with TLC. If you know a bit about fairness, then you saw this from a mile away, but `Prop` actually **doesn't hold for this spec**.

Ask TLC to check this, and you'll get the following counterexample:
```
<<
[
 _TEAction |-> [
   position |-> 1,
   name |-> "Initial predicate",
   location |-> "Unknown location"
 ],
 x |-> FALSE,
 y |-> FALSE
],
[
 _TEAction |-> [
   position |-> 2,
   name |-> "B",
   location |-> "line 14, col 6 to line 16, col 19 of module simple_liveness"
 ],
 x |-> FALSE,
 y |-> TRUE
]
>>
```
We take a `B`-step, and then we stutter forever. 

>[!REMINDER]- Stuttering Steps
>A stuttering step is a step that keeps the variables the same. Essentially, it means that the spec refused to take a step (even if an enabled step exists).
>Why do we need it? The reason is spec composition.
>
>If we have two specs, `Spec1` and `Spec2`, and we combine them somehow to create a new spec `Spec3`, we suddenly have steps that change variables in one spec but leave them unchanged in another, unless we want to amend each step in each specification with an extra `UNCHAGED` statement, which seems tedious.
>If specifications allowed stuttering steps however, this problem would go away. If `Spec1` took a step, `Spec2` can stutter and vice-versa.
>
>The problem here is that, as far as TLC and TLAPS are concerned, there is no difference between an action that exists to facilitate model composition, and one that is actually part of the logic of the model. If we allow a model to stutter, why shouldn't it stutter *forever*?
>
>This is where our troubles begin.

>[!REMINDER] Weak/Strong Fairness
>We have abused the word "step" slightly in the discussions that we had until now. If $A$ is some action, then we usually deal with it in two forms:
>- A step may refer to $[A]_{vars}$ , i.e. a step that might satisfy the $A$ action but otherwise would leave $vars$ unchanged. I may refer to this as a *stuttering* *$A$-step*.
>- A step may also refer to $\langle A\rangle_{vars}$, i.e. a step that satisfies $A$ *AND* changes $vars$. I may refer to this as a non-stuttering $A$-step.
>
>With this settled, here is the definition of Weak and Strong Fairness:
>- *Weak Fairness:* ${\huge\diamond}\square~\text{ENABLED}~A \implies \square{\huge\diamond}\langle A \rangle_{vars}$
>- *Strong Fairness:* $\square{\huge\diamond}~\text{ENABLED}~A \implies \square{\huge\diamond}\langle A \rangle_{vars}$
>  
>The plain English version of these would be:
>- Weak Fairness of $A$ implies that if $A$ remain continuously enabled then infinitely many non-stuttering $A$-steps must happen.
>- Strong Fairness of $A$ implies that if $A$ is infinitely often enabled then infinitely many non-stuttering $A$-steps must happen.
>
>The emphasis on *non-stuttering* is crucial, since if an action never changes certain variables but is continuously/infinitely often enabled, then fairness cannot be used to specify liveness about such variables (see [this example]([tla+ - PlusCal: Why does fair algorithm still stutter? - Stack Overflow](https://stackoverflow.com/questions/55128505/pluscal-why-does-fair-algorithm-still-stutter)) for how this can manifest)

So as the discussion above shows, we need fairness conditions, at least for `Next`.
```
Spec == Init /\ [][Next]_vars /\ WF_vars(Next)
```
And with this, the spec finally passes. We can now make an attempt on the proof.

>[!NOTE] Using `ENABLED`
>In TLA$^+$ syntax, using $\text{{\tiny ENABLED}}~A$ is the boolean expression that determines when taking some $A$-step is possible.
>TLAPS can optionally expand this expression to its actual value. TLAPS defines a token, `ExpandENABLED` for this which we can pass to a `BY` proof.

Now, how to prove `<>(x = TRUE)`? One way is a proof by contradiction. Note that ${\huge\diamond}P \equiv {\neg\square\neg}P$, thus $\neg{\huge\diamond}P \equiv \square\neg P$.

>[!BUG] About Proof By Contradiction
>For whatever reason, TLAPS is extremely bad about Proof by Contradictions. I am not familiar with the inner workings of TLAPS to know why this happens, but I could never get it to work reliably by deriving contradictions from contexts.
>To make this more concrete, assume we want to prove $P \implies Q$. A proof by contradiction would have that $P \wedge \neg Q \implies \text{FALSE}$.
>This can be written in TLAPS, among other ways, like the following:
>- `<n> SUFFICES PROVE P /\ ~Q => FALSE`
>- `<n> SUFFICES ASSUME P, ~Q PROVE FALSE`
>
>I have had many cases where the second form fails. For whatever reason, TLAPS has trouble accepting contradictions in its assumption context. Thus try to keep using the first form.

For our problem, assuming the negation would mean that we can add an extra hypothesis that `[]~(x = TRUE)`. This would mean that action $A$ remains continuously enabled, and thus a non-stuttering $A$-step eventually happens, or does it ... ?

We have only specified fairness for $Next$, why would it translate to fairness in $A$ as well? The reason is because every $A$ or $B$ step actually disables it when taken. Thus if $B$ step was taken, then $Next$ cannot remain enabled without enabling $A$.
Thus, for this example, Weak Fairness of $Next$ is enough for us.

So our proof by contradiction would look like this:
```
THEOREM Spec => Prop
<1> SUFFICES ASSUME []TypeOK PROVE Spec /\ []~(x = TRUE) => FALSE
    BY TypeOK_inv, PTL DEF Prop
<1> QED
```
Now, if `[]~(x = TRUE)`, it means that at each state, we have both `~(x = TRUE)` and `~(x = TRUE)'`. Thus, it essentially means that a non-stuttering $A$-step must never happen. However, `[]~(x = TRUE)` means that $A$ is also always enabled. Thus to satisfy this, we would have to violate weak fairness.

Thus, the sketch of our proof would look like this:
- Show that 
خسته شدم بابا بسه دیگه ....