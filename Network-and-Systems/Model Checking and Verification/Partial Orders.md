**Source:** [Partial Order Methods for Software Verification](https://patricegodefroid.github.io/public_psfiles/thesis.pdf)

In the context of model checking software, the goal has always been to try and find corner cases where certain properties and invariants fail to hold, and use that to diagnose what should be changed that can allow one to fix them.

Such techniques are referred to as *State-Space Exploration* methods, where upon knowing possible actions on any state (either via a symbolic method or an explicit specification) one attempts to find all chains of valid behaviors (*i.e.* sequence of states) that a system can exhibit.

>The main limit of state-space exploration verification techniques is the often excessive size of the state space due, among other causes, to the modeling of concurrency by interleaving.

We have also had similar issues when developing the specification for [[ZENITH]], where even for a small network size of a only a few switches and a few instructions, the verification could take multiple hours or just die because of memory constraints.

The main thing to note here, however, is that one need not evaluate *all* executions, only those that end up in different states.
