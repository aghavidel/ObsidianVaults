# TLA+
## Intro.

Created by [Leslie Lamport](http://www.lamport.org/) , TLA+ (In papers, you write it as TLA$^+$) is a high-level language that aims to formally verify the *correctness* of algorithms that run over distributed systems or have some concurrent element built into them.

TLA+ isn't going change your algorithm or rewrite it in a better way, in truth, the bulk of TLA+ is the following things:

- A language that describes a *specification* for a system and introduces *invariances* that such a system **MUST** follow for *any* execution.

- A *model checker* that tries to actually check whether or not the specification is consistent. Of course, the space that the model checker can search is *finite* whereas as the true size of the execution space is *infinite*. 
  But we can at least be more confident that our model works. Especially from an engineering perspective, this would be fine on its own.

> [!TLDR]
TLA+ accepts an specification from you and a set of rules that it must follow, and tells you, with a certain degree of certainty, how much you missed the mark.


## How Does it Work?

I honestly don't understand a word of it yet.

TLA+ stands for ***Temporal [[Logic of Actions]], Plus***, which is Lamport's extension over the more general term of [[Temporal Logic]] . He has a landmark paper on the matter which you can find [Here](https://www.hpl.hp.com/techreports/Compaq-DEC/SRC-TN-1994-001.pdf)

Since the early days of research into distributed systems, the question of *how* one would actually characterize the execution of these systems and verify their behavior was already a nagging problem for researcher. 

While people already had guidelines that defined and controlled the logic of concurrent systems (especially, computer scientists who were trying to create operating systems), they were never actually *formally verified*, they merely presented themselves as non-trivial solutions to problems, which could be verified by brute forcing the execution space.

While the execution space is indeed almost always *infinite*, not every execution is meaningfully distinct from the other, and an equivalence relation can usually be found that completely partitions the execution space. 
At that point, verification can be done by verifying the execution of a *finite* set of equivalence classes.

This is how most concurrent systems, and solutions to problems like [[Dining Philosophers]] were found at the time and were accepted formally. But actually *finding* that relation that breaks the execution space was already a non trivial challenge. Turning it into a formal description is even more daunting.

Lamport however, tried to keep things rather simple in his first specification and instead allowed the execution space to be incrementally searched until the model specifier was satisfied with its current specification.

What he ended up creating was [[Temporal Logic Action]] which became the base of TLA+ specification.


## Language Concepts

While TLA+ can be created by translating [[PlusCal]] (a higher level version of TLA+), it is necessary to know at least a bit of TLA+ by it's own to understand PlusCal well. Here we will go over some simple concepts about what we actually write in TLA+.

### Data Types

TLA+ sees everything as a *set*. Even things like `1` or `"abc"` are considered sets, though their contents may not be necessarily what they look like, so doing something like checking for the expression $1 \in \text{"}abc\text{"}$  will still produce errors.

That aside though, TLA+ uses pretty familiar data types for the most part. Numbers and Strings are written like most programming languages and common operators are still used:
- Arithmetic operators: `+`, `-`, `*`, `%` (don't worry about division), we also have exponentiation with `^`.
- Comparison: `>`, `<`, `=<`, `=>` 
- **EQUALITY** and *not assignment*: `=` 
- **NOT EQUAL** : Either `#` or `/=`

There are also boolean types `TRUE` and `FALSE`.

>[!WARNING] A Warning About Strings
>TLA+ does not have (nor it particularly needs) anything to do with strings. There is no `Char` type in TLA+ and strings are not treated as pointers or arrays.
>
>If your algorithm is supposed to use strings, then you might wish to create a one-to-one mapping between strings and integers and start using that instead of strings. Also, don't treat strings as their own separate *type*. 
>
>If you were to evaluate something like `1 = "123"` in TLA+, the TLC model checker will complain, simply because strings are **NOT** supposed to be compared to integers, this evaluation here will not return `FALSE` as it would in programming languages.

Variables are declared at the beginning of the program with the `VARIABLE ...` prefix, like:

```
VARIABLE a, b, c, ...
```

Sometimes, it is also needed that a variable is assigned a value once and it should never change, for these, we can use the `CONSTANT` keyword instead.

>[!WARNING]
>Redefining variables is not allowed in TLA+.

There are also predicates, which can be declared with a name and then assigned to with `==`, we'll give some examples along the way, but for the concrete definition, go to [[#Predicates]].

### IF-THEN-ELSE

They have fairly intuitive syntaxes:

```
Abs(x) == IF x < 0 THEN -x ELSE x
```

>[!IMPORTANT]
>Expressions always have to be assigned to *something* in the end, so you can't omit the `ELSE` statement above like you can in coding.


### Global And Local Definitions

Definitions are either global, meaning that they can be accessed throughout all of the spec *after the line that defines them*, or local, which means that they can be limited to a certain scope and be invisible beyond that.

Global definitions are rendered with $\triangleq$ and are written as `==` in ASCII. We saw these above.
Local definitions can be done with the `LET-IN` syntax, where `LET` starts the definitions and `IN` starts a scope that can actually use that definition.

Here is a example:

```
MaxOfThreeNumbers(a, b, c) == 
	LET
		MaxOfTwoNumbers(a, b) == IF a > b THEN a ELSE b
	IN
		MaxOfTwoNumbers(MaxOfTwoNumbers(a, b), c)
```

### Arrays

Arrays have a rather arcane way of setting up, owing to the fact that they are mathematical constructs here rather than memory blocks. Arrays have an *index set*, and an *expression*. Using respective symbols of $I$ and $e(.)$ for all of these, an array in TLA+ is defined as:
$$
[v \in I \rightarrowtail e(v)]
$$
Which is actually a *map* or a function, rather than an array in coding. For example, the array:
$$
sqr \triangleq [i \in 1\;..\;12 \rightarrowtail i^2]
$$
Returns the list $[1, 4, 9, 16, 25, ..., 144]$ . The symbol $\rightarrowtail$ is written as `|->` in ASCII.

This is much more limiting than the notation of arrays in programming languages, but it also has a benefit, that TLA+ can actually *check* what kind of array we are using at each step. More specifically, the expression:
$$
[I \rightarrow D]
$$
Means "The set of all arrays indexed by the set $I$ that maps into the domain $D$". So the $sqr$ array for example can be checked with the following expression:
$$
sqr \in [1\;..\;20 \rightarrow 1\;..\;400]
$$
Which would return `TRUE`.

Similarly, we can filter over a set using a predicate, say we have a set $S$ and we have a predicate $P$, the set of values in $S$ that satisfy $P$ can be written as:
$$
[v \in S: P(v)]
$$
>[!NOTE]
>As you can see, TLA+ is indices start from 1.

### Tuples

Tuples are essentially sets of various constants or variables. They are written with a syntax like `<<t1, t2, ..., tn>>` and are rendered as:
$$
\langle t_1, t_2, ..., t_n\rangle
$$

### Predicates

Probably the heart of TLA+ comes from what it borrows from [[Temporal Logic Action]], i.e. using predicates that describe actions over states using simple [[First Order Logic]] formulas.

TLA+ uses pretty much any operator that you would use in first order logic, the same way that you would do on paper, with some hoops to jump.

All free variables in these formulas must be declared beforehand, but bounded variables used with quantifiers can be declared inline with the formula by just writing them. Predicates may be assigned a name, like:
$$
Predicate \triangleq x \wedge y
$$
would declare a predicate `Predicate` throughout the program. These predicates can also be broken down into smaller pieces, making sure that they are defined before used, like:

$$
F_1 \triangleq x \wedge y \quad F_2 \triangleq z \wedge y
$$
Then we can have:
$$
Predicate \triangleq F_1 \vee F_2
$$

As for how we would write them in ASCII, we have:

- Definition:
	- Defining a predicate with $\triangleq$ is done with `==`
- Logical Operators:
	- Conjunctions ($\wedge$) is written as `/\`
	- Disjunction ($\vee$) is written as `\/`
	- Negation ($\sim$) is written as `~`
- Quantifiers and Temporals:
	- Existential quantifier ($\exists$) is written as `\E`
	- Global quantifier ($\forall$) is written as `\A`
	- "Always" ($\square$) is written as `[]`
	- "Eventually" ($\diamond$) is written as `<>`
- Set operators:
	- Is in ($\in$) is just `\in`
	- Is subset of ($\subseteq$) is `\subseteq`
	- Union ($\cup$) is `\cup`
	- Intersection ($\cap$) is `\cap`
	- Set subtraction is just `\`

Predicates can take inputs as well, like:
$$
F_1(x) \triangleq x \wedge y
$$
And then after this we can have things like:
$$
F_2 \triangleq F_1(z) \vee w
$$
Which is equivalent to $z \wedge y \vee w$ .

Predicates describing temporal actions involve primed and unprimed variables, as described in [[Temporal Logic Action]], so we can have for example:
$$
Pred \triangleq stateVar' = F(stateVar)
$$
Which updates the variable `stateVar`.

We can also chain some of them, though beware that they are always commutative. So for example a simultaneous assignment to `stateVar1` and `stateVar2` can look like the following:
$$
\begin{align}
Pred \triangleq & \wedge stateVar1' = F_1(stateVar1) \\
& \wedge stateVar2' = F_2(stateVar2)
\end{align}
$$
There are also some special predicates, like the $\text{UNCHANGED}$ predicate that takes a tuple of variables and says that they have not changed. So for example writing the predicate:
$$
Pred \triangleq UNCHANGED\;\langle stateVar1, stateVar2\rangle
$$
Is the same as writing:
$$
\begin{align}
Pred \triangleq & \wedge stateVar1' = stateVar1 \\
& \wedge stateVar2' = stateVar2
\end{align}
$$
As you can see, we have this "Bullet-Point" way of chaining expressions, for example the expression `A /\ (B \/ C) /\ (D \/ E)` is written as:

```
/\ A
/\ \/ B
   \/ C
/\ \/ D
   \/ E
```

That first `/\` really isn't needed, it's just for syntactic sugar. **This is probably the only place that white space matters to my knowledge!** 

To show this, consider that we wrote

```
/\ A
/\ \/ B
\/ C
/\ \/ D
\/ E
```

This is the expression for `A /\ B \/ C /\ D \/ E`  instead!

### Sequences

Finite sequences can be described in TLA+ with the syntaxes that we used for arrays, but since they come of very often, they have their own module within TLA+ called `Sequences` (see [[#Importing Specs]]).

Within have some extended syntax for these.
- $Head$ returns the first element of a sequence
- $Tail$ returns everything except the first element
- We have a concatenation operator that joins two sequences, using `\o` or $\circ$ when rendered, so we have for example:
$$
\text{IF} \quad seq \neq \langle\rangle \quad \text{THEN} \quad seq = \langle Head(seq) \rangle \; \circ \; Tail(seq)
$$
- We have an operation for appending things to the end of a sequence, which literally means:
$$
Append(seq, e) \triangleq seq \;\circ \; \langle e \rangle
$$
- We have the $Len$ operator that returns the length of a sequence, and so the domain of the sequence can be defined as $1 .. Len(seq)$ if we let $1..0 = \{\}$ which TLA+ does.
- The set $Seq(S)$ is the set of sequences that are created using the elements of the set $S$, for example:
$$
Seq(\{2\}) = \{\langle\rangle, \langle 2\rangle, \langle 2,2\rangle, ...\}
$$
- For Cartesian product of sets or sequences, we use `\X` 

>[!EXAMPLE] Defining $Remove(i, seq)$
>We define an operator for removing the element $i$ of a sequence $seq$, this would be paramount for when we want to implement protocols that let messages be lost along the way.
>
>One way would be like this:
>$$
>\begin{align}
>Remove(i, seq) \triangleq
>\quad [j \in 1..Len(seq)-1 \rightarrowtail \; \text{IF} \; j < i \; & \text{THEN} \; seq[j] \\
>& \text{ELSE} \; seq[j+1]]
>\end{align}
>$$
 

### Records

A record, corresponds almost to a C struct. It maps keys to values (once again, more like a function than a programming struct) and can be accessed similarly to a C struct.

For example, given keys `SQN`, `NXT` and `UP`, we can create a struct called `SND` to keep the record of some TCP send buffer variables like the following:
$$
SND \triangleq [SQN \rightarrowtail 100, \; NXT \rightarrowtail 300, \; UP \rightarrowtail 0]
$$
And we can access `SQN` with $SND.SQN$, which is in fact, completely the same as $SND[\text{"}SQN\text{"}]$, just a bit more compact.

Like arrays, sets of records can also be described. For example the syntax:
$$
[SQN \rightarrowtail 0..200, NXT \rightarrowtail 0..100, UP \rightarrowtail 0..1]
$$
Describes all records whose `SQN` field is in `[0, 200]` and whose `NEXT` field is in `[0, 100]` and whose `UP` field is in `[0, 1]`. With this definition, the `SND` record is *not* in this set.


### Importing Specs

Usually, one spec in the TLA+ model specifies *what* should be done, and another spec actually *implements* it (see for example, [[TLA+ Examples# Two Phase Commit]]).

It is apparent that we need some way of importing specs and statements within specs, so one way we can do that would be to use the `INSTANCE` syntax.

If the specs `SPEC1` and `SPEC2` are within the same folder, then we import everything in `SPEC1` into `SPEC2` by writing:

```
SP1 == INSTANCE SPEC1
```

In the body of `SPEC2`. 

If `PredInSPec1` is some predicate defined in `SPEC1` we can now reference it using the bang syntax like below:

```
SP1!PredInSpec1
```

There are also some other default specs that can be used in the program. These use the `EXTENDS` notation.

For example, the set of all natural numbers (`NAT`) is defined in the spec `Integers`, and so we can use it like the following:

```
EXTENDS Integers

VARIABLES x

Predicate == x \in Nat
```

Where `Predicate` is TRUE only if `x` is a natural number.

### The `CHOOSE` Syntax

The expression written as $\text{CHOOSE} \; v\in S : P$ means "If there is a $v$ in such as $P$ is true, return it". 

>[!WARNING] Multiple Choices Of $v$
>If there can be a whole set of such values, there really isn't a definitive answer as to which one it will be chosen.
>
>We either *do not care* as long as $P$ remains true, or we are sure that there will always be only one such value.

>[!FAIL] A Common Mistake
>If model parameters are symmetric, they **CANNOT** appear in a `CHOOSE` statement.

### Comments

```
\* Inline comment here ...
(*
	Multiline comment 
	here ...
*)
```

As most TLA+ specs might be pretty-printed with `PdfLatex`, there are two more variations on comments:

- **Boxed Comments** are enclosed completely in multiline delimiters like below:

```
(***************************)
(* This is a boxed comment *)
(* It is pretty ...        *)
(***************************)
```
  
  They are printed into the resulting `PdfLatex` document.

- **Separators** which just separate parts of the spec for readability and printed as simple horizontal lines in the pretty-printed document:

```
-----------------------------------------
```


## Implementing A Spec

Once the algorithm has been specified, it needs to be evaluated, showing that it matches certain specs and invariants.

The general structure of a TLA+ specification is:
- The spec *should* specify constants (i.e. parameters of the algorithm) and variables.
- The spec *must* specify the assumptions for each constant.
- The spec is *recommended* to type check variables.
- The spec *should* specify how the algorithm starts (the $Init$ predicate) and what it can do to transition in it's state machine (the $Next$ predicate)
- The spec *must* then specify what exactly should it adhere to.

You can find a few detailed examples in the [[TLA+ Examples]] or a much bigger set of them at the [TLA+ GitHub](https://github.com/tlaplus). 

### How To Actually Check The Spec

You should refer to the [[TLC Model Checker]] note first, but the main idea of the specification verification comes from what we discussed about [[Temporal Logic Action]].

>[!REMINERD]- How Temporal Logic of Actions, Verifies Specs.
>A specification, at it's core, specifies valid *behaviors*, which were defined as sequences of states. If the spec can be expressed as a temporal logic formula $F_S$ and the properties or invariants as a formula $F_{inv}$, then the model consistently implements the spec if and only if we have:
>$$
>F_S \longrightarrow F_{inv}
>$$
>For all behaviors.

For a TLA+ model to be a verified implementation of a spec, we should have the following:

>[!IMPORTANT] Verifiability Of A Spec
>A spec is verified as a consistent implementation of an algorithm, if and only if for all behaviors $\sigma = s_1, s_2, ..., s_n$ we have:
>- $Init$ is true for $s_1$.
>- $Next$ is true for all steps $(s_i, s_{i+1})$ for all $i \geq 0$.

So, in the form of temporal logic, the specification can be expressed as:
$$
Spec \triangleq Init \wedge \square Next
$$
And so, if $Spec$ is true on all behaviors, the model is verified.

TLA+ makes it a bit more complicated by writing it this way:

>[!IMPORTANT] TLA+ Specification Formula
>A specification with the initial predicate $Init$, the next state formula $Next$ and declared variables $v_1, ..., v_n$, is expressed as:
>$$
>Spec \triangleq Init \wedge \square [Next]_{\langle v_1, ..., v_n \rangle}
>$$

The formula above is written like this in ASCII:

```
Spec == Init /\ [][Next]_<<v1, ..., vn>>
```


>[!FAQ]- Why The Complication?
>The extra syntax allows for *stuttering steps*, which are steps where a program just does not do anything.
>
>It is not hard to see that without stuttering steps, some of our theorems just do not hold, simply because the spec and in the invariant may put different conditions on *the same* step. 
>
>For an example, consider the  Two Phase Commit spec in our [[TLA+ Examples]], the theorem for this spec asserts that $TPSpec \longrightarrow TCSpec$, but if you look at the modules, you will see that:
>- $TPSpec$ allows for $TMAbort$ steps that leave the variable $rmState$ unchanged.
>- $TCSpec$ however has a $TCNext$ action for each step, which *definitely* changes $rmState$.
>
>So given these, the theorem *cannot* be true!
>
>The solution would be to let the variables remain unchanged. 
>This does not change the meaning of the $Next$ predicate, as it still holds true for every step of any behavior, even though many of these steps can be vacuous. So we can write:
>$$
>TCSpec \triangleq TCInit \wedge \square (TCNext \vee \; \text{UNCAHNGED} \quad rmState)
>$$
>
>This, is equivalent with extra syntax for $TCNext$, where we write $[TCNext]_{\langle rmState \rangle}$.


These can be plugged into the TLC model checker in the `Temporal Formula` part.

Any state formula that is true for all behaviors is called a *Property*, and can be added to the model checker.

For example, type checking predicates like $TypeOK$ are a property, they can be verified by specifying that $\square\; TypeOK$ is a property, or just say that it's an invariant.

To make this easier, specs should write a `THEOREM` at the end, which is basically just that the spec should yield the invariants, for example:

```
THEOREM Spec => TypeOK
THEOREM Spec => SomeConsistencyPredicate
```

Formally, any state expression that is true for all behaviors, is called a *theorem*.

And to check them, we just add `TypeOK` and `SomeConsistencyPredicate` as model properties in the model checker. We then say that  ***$Spec$ Implements $TypeOK$***.


## Safety, Liveness and Fairness

These are probably the most important parts of TLA+, how would we express them?

### Safety

Safety specifies what *may* happen, meaning that if a behavior $\sigma$ violates a formula $F_{safe}$ that expresses safety, then at some point during the execution of that behavior, the formula will return `FALSE` and **nothing after that point can correct it**.

We have already expressed safety by specifying what we want with invariants, if a behavior was to violate the temporal formula $Init \wedge \square \; [Next]_{vars}$ it must either:
- Not let $Init$ be true at the first state.
- Have a step that neither leaves $vars$ unchanged nor satisfies $Next$.

And of course, nothing after this violation can fix it!


### Liveness

Liveness specifies what *must* happen, meaning that if a behavior wants to adhere to this spec, it **CANNOT** violate it at any point, no matter whether or not the rest of the behavior can make the formula expressing it true again.

For example, we can say "there is some state in the spec that allows for $x = 5$", meaning that eventually, $x =5$ happens at some state.

Termination is probably the most well-known liveness property, since it applies to every sequential program. Concurrent systems are a bit more complicated though.

Usually this is expressed as $A \rightsquigarrow B$ which means that "A eventually leads to B", which can be written as `A ~> B`.


### Fairness

Fairness in temporal logic is expressed as either *weak* or *strong*:

- **Weak Fairness** for an action $A$ asserts that "$A$ cannot remain enabled forever without an $A$-step happening". We write it as `WF_vars(A)` and rendered as $WF_{vars}(A)$, where $vars$ is the set of all variables in the spec.
- **Strong Fairness** for an action $A$ asserts that "$A$ cannot be repeatedly enabled forever without an $A$-step occurring". We write it as `SF_vars(A)` and rendered as $SF_{vars}(A)$ like above.

To express fairness, the spec is usually supplemented with a formula $Fairness$, which is a conjunction of either $WF(A)$s or $SF(A)$s where each $A$ is some sub-action of $Next$, so with this, the final spec would be:
$$
Spec \triangleq Init \wedge \square\; [Next]_{vars} \wedge Fairness
$$
>[!FAQ]- Why Again That $vars$ subscript?
>If you wish to check fairness, you **should** remove stuttering steps, since there is no way to actually say whether or not a stuttering step *has* occurred, let alone checking that it is happening after a formula is forever enabled.
>
>If we were to express this, we would write that:
>$$
>A \wedge (vars \neq vars')\; \text{Continuously enabled}  \rightarrow (A \wedge (vars \neq vars'))-\text{step eventaully occurres}
>$$
>
>And that is exactly what $WF(A)_{vars}$ means.

