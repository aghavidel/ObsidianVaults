# Intro.

PlusCal is s higher level way of specifying [[TLA+]] code. While it has the same functionality and does not offer anything that TLA+ can't already do, it has the benefit of being much more:
- Readable
- Compact
- Less prone to errors

We assume that we are fully familiar with TLA+ as we continue to read this note, otherwise, some of the terms used might not be very clear, so if that is not the case, go back and read the TLA+ note.

## An Example

PlusCal is based off TLA+, and the data types and most of the syntax is the same. Modules are still written between a start and end delimiter like the following:

```
=============== MODULE ===============

           <CODE GOES HERE>
			
--------------------------------------
```

The main addition of PlusCal comes in the form of *algorithms*. See the followingL

```
(********

--algorithm AnyName {
   variable x = 1 ;    
   {  
     x := x + 1 ;
     print x
   }
}

********)
```

Here, `AnyName` can be any valid TLA+ identifier. It should be unique within the scope of the specification of course. As you can see:

- Algorithms are scoped with curly braces like many other languages.
- Algorithms, similar to TLA+ specifications need to declare variables first.
- Variables can be initiated right at the beginning of the declaration.
- You can also see semicolons (`;`) being used to separate statements. 
- Assignment (and NOT declaration) is done with `:=`.
- Print can be used to output the value of it's argument like any other programming language. 

>[!FAQ] Where is `print x` semicolon?
>The semicolon in front of `print x` is missing, but that is not a problem. The *last statement* of a scope can be left without a semicolon without any problem. There is also no problem if you would like to still put it pout of habit.

You may also have noticed that everything we wrote is actually a comment! What's up with that?

### How To Run PlusCal Code

PlusCal needs to be *interpreted*, which turns it into TLA+ code that [[TLC]] can then understand and evaluate. If you run the piece of code above, it would not do anything at all!

TLA+ toolbox provides a function that allows you to interpret the PlusCal code and turn it into simple TLA+ code. This function is under `File > Translate PlusCal Algorithm`, but it is easier to use `ctrl + T` shortcut instead.

Doing so, spits out a TLA+ code block right under the PlusCal code that we written. We'll get to it later, but just note that for the most part, this code can be run like any other TLA+ specification. 

So, we just create a model and evaluate it. Doing so just outputs the value `2` to the TLC console.

### Another Example

Let's do this:

```
--variable x = <<1, 2, 3>> , y = x ;    
{  
  x[3] := x[2] + 4 ;
  print x ;
  print y ;
}
```

Running this, produces the output `<<1, 2, 3>>` for both x and y first. But on the next state, the value of x will be `<<1, 2, 6>>` and y will remain **unchanged**.

Note that these are still interpreted as temporal formulas! So x and y equal together only during the beginning (i.e. the initial state).

Let's take the time to look at what TLA+ actually outputs:

```
VARIABLES x, y, pc

vars == << x, y, pc >>

Init == (* Global variables *)
        /\ x = <<1, 2, 3>>
        /\ y = x
        /\ pc = "Lbl_1"

Lbl_1 == /\ pc = "Lbl_1"
         /\ x' = [x EXCEPT ![3] = x[2] + 4]
         /\ PrintT(x')
         /\ PrintT(y)
         /\ pc' = "Done"
         /\ y' = y

(* Allow infinite stuttering to prevent deadlock on termination. *)
Terminating == pc = "Done" /\ UNCHANGED vars

Next == Lbl_1
           \/ Terminating

Spec == Init /\ [][Next]_vars

Termination == <>(pc = "Done")
```

As you can see, a new variable called `pc` is created. This is essentially TLA+'s *program counter*, it uses it to map your algorithm to temporal formulas. As you can see, it requires one tick, since only a single next state is enough to describe what we are doing. These should look familiar to anyone who has worked with TLA+ a bit.


## Algorithms

We now go over the general structure of algorithms in PlusCal. 

### Pre/Postconditions

Often, we will have constraints on the input/output of an algorithm. Constrains on the input are called *Preconditions* and constrains on the output are called *Postconditions*. To write these conditions in the specification body, we write them as assertions during the beginning and end of the algorithm.

For example, let's say we wish to write a program that computes the maximum element of a tuple. A postcondition would be that the result should be in the tuple and it must be larger or equal to all the other elements, and a precondition can be a lower bound on the values of the tuple (like everything is larger than -99999).

So we could write:

```
--algorithm TupleMax {
   variables inp = <<1, 3, 2>>,  max = -99999, i = 1 ;    
   { 
     assert  \A n \in 1..Len(inp) : inp[n] > -99999 ;
     
     ...
     
     assert    (\E n \in 1..Len(inp) : max = inp[n])
            /\ (\A n \in 1..Len(inp) : max >= inp[n])  
   }
}
```

Now we come to one the main feature of PlusCal in comparison to TLA+. Loops and Conditionals!

```
while (i =< Len(inp)) {
	if (inp[i] > max) { max := inp[i] } ;
	i := i + 1 ;
} ;
```

Take note that the curly braces for the `if` and `while` **still end with a semicolon**.

The -99999 value is quite arbitrary and ugly, it needs to be declared as a constant. In PlusCal we can do this with the `ASSUME` statement. Say:

```
--algorithm TupleMax {

	ASSUME minValue \in Int ;
	variables inp = <<1, 3, 2>>, max = minValue ;
	assert \A n \in 1..Len(inp) : inp[n] > minValue ;
	
	while (i =< Len(inp)) {
		if (inp[i] > max) {max := inp[i]} ;
		i := i + 1 ;
	} ;
	
	assert /\ (\E n \in 1..Len(inp) : max = inp[n])
		   /\ (\A n \in 1..Len(inp) : max >= inp[n])
}
```