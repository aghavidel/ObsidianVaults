# Intro.

The Bellman-Ford algorithm is a general algorithm to solve the Shortest Path problem over a weighted digraph.

>[!REMINDER] Single-Source Shortest Path Problem Over a Digraph
>Given a digraph $\mathbf{G} = (V, E)$ and a source vertex $v_s$, find one of the shortest paths from the source vertex to any other vertex.
>
>Note that in general, each edge can have a negative weight.


This algorithm is the slower, but more general cousin of [[Dijkstra's Algorithm]] used for the same problem. It is more general, since it can handle the case of negative weights in the graph, and even check whether or not such a minimum path actually exists.

>[!WARNING] Negative Cycle Problem
>If a cycle exists in the graph $\mathbf{G}$ that has a total weight of $-w$ for some $w>0$, and can be traversed (i.e. it is reachable from the source and does not contain the destination) then there is no shortest path. 
>
>Simply because if any path between to arbitrary nodes $v, u$ was declared as optimal, a better path can be created by taking a detour halfway on the aforementioned cycle, and loop to our heart's contend until the weights become less than the original "optimal" path.


## Intuition

Similar to Dijkstra's algorithm, this algorithm performs a series of relaxations over available distance measures, and converges overtime to optimal values. In Dijkstra's algorithm, the relaxation is performed greedily, but here, every single edge during a single round, instead the number of rounds is bounded by the number of vertices (it's $|V| - 1$ to be exact).

So we first overestimate the distance and then we iteratively lower that distance to it's optimum value. Whether or not we actually arrive at that point is not trivial, and requires proof. But regardless of that, this step (the relaxation) can be done like the following:

Let $\mathbf{G} = (V, E)$ and let $v \in V$, let us keep the distance of each node from $v$ in an array called $D$, then:

```python
def relaxation(v, E):
	for each edge (u, v) in E:
		relaxed_distance = G.weight(u, v) + D[u]
		if relaxed_distance < D[v]:
			D[v] <-- relaxed_distance
```

We hope that repeated applies of this relaxation will find any "backdoor" route to $v$ that we might have missed.

It turns out that we need exactly $|V| - 1$ relaxations to achieve this.

## The Algorithm

Just relax the distances $|V| - 1$ times.

```python
==========================
	Bellman-Ford
==========================
'''
Given G := (V, E)
Given some s in V as source
'''
--------------------------

'''Initialize'''
global D := list()
for u in V:
	D[u] <-- inf
D[s] <-- 0

'''Loop'''
for n in range(len(V)-1):
	relaxation(v, E)

return D
```

It's not really over yet, if we perform this operation till the end, what we'll have is a list of distances from the source node, which by itself isn't enough to get a path.

Here, Bellman's Optimality Principle will help us:

>[!IMPORTANT] Bellman's Optimality Principle
>Given an optimum path $P$ from vertex $u$ to vertex $v$ in a weighted graph $\mathbf{G} = (V, E)$, then for each intermediate node $w$ on this path, the segments $P(u \rightarrow w)$ and $P(w \rightarrow v)$ are also optimum paths from their respective source nodes to their destinations.

It is not hard to see why this would be the case, as anything other than this will lead to either that path not being the optimal path, or such a path not even existing in the first place. 

One consequence of this fact is that if we were to trace all of these optimal paths on the graph and remove every other edge from the graph, we'll end up with a DAG, or just a tree, if the starting graph is not directed.

Why? let's call this subgraph $\mathbf{G}^*$ .
- $\mathbf{G}^*$ is certainly connected, as there is a path to each node by definition.
- $\mathbf{G}^*$ can be modified to have no cycles.

The first one is obvious, the second one needs a proof:

>[!PROOF]- Why $\mathbf{G}^*$ can be acyclic.
>Assume a cycle $\mathcal{C}$ exists in $\mathbf{G}^*$. 
>
>The cycle must have been formed through the concatenation of several sub-paths of our optimal paths from the source node $s$ to all other nodes in the graph. The cycle may not include the source itself, since if any path returns to the source, it cannot be optimal (unless a negative cycle exists, which means that all of this analysis is pointless, so let's assume that is not the case).
>
>Let $v$ be the nearest node of the cycle, so set:
>$$
>	v = \underset{u \in \mathcal{C}}{\mathrm{argmin}} \; d(u)
>$$
>This means that either $v$ is one hop away from the source, or any intermediate node on the path to $v$ is not on the cycle. Call this path $P(s \rightarrow v)$
>
>Now since there is a path from $v$ back to itself, it means that there is some path that starts from the root, comes from another way to $v$ and then branches out to reach it's destination. This implies that there must be some equally optimal path from the source to $v$ other than the one that does no involve the cycle.
>
>We can simply just modify this path to not take this unnecessary detour and just replace the sub-path that goes to $v$ with $P(s \rightarrow v)$, thus removing this cycle.

This is good, since DAGs and trees can be described by just having a list of *predecessors* for each node, and using that list, we can find the path to each node by backtracking from the destination.

Who should be the predecessor? According to our discussion, the predecessor should be the node immediately behind the destination in the optimal path taken. So we can find this node during the relaxation step.

So we can do:

```python
==========================
	Bellman-Ford
==========================
'''
Given G := (V, E)
Given some s in V as source
'''
--------------------------

'''Initialize'''
global D := list()
global P := list()
for u in V:
	D[u] <-- inf
	P[u] <-- None
D[s] <-- 0

def relaxation(v, E):
	for each edge (u, v) in E:
		relaxed_distance = G.weight(u, v) + D[u]
		if relaxed_distance < D[v]:
			D[v] <-- relaxed_distance
			P[v] <-- u

'''Loop'''
for n in range(len(V)-1):
	relaxation(v, E)

return D, P
```


## Proof of Correctness

>[!NOTE] Lemma 1
>After $i$ repetitions of the loop, $D[u]$ is either infinite, or equal to at most, the length of some path from $s$ to $u$.

>[!IMPORTANT]- Poof
>It's a simple induction. 
>
>For the base case, it is obvious. When $i=0$, the distances to all other nodes are infinite, the distance to $s$ is of course zero, corresponding to the zero hop path from the source to itself.
>
>For the inductive step, assume the lemma holds up until step $i$. The update will have a relaxation step over all nodes, so for any node $v$ and any neighbor $u$, let the weight of the edge connecting the two be $w$. In this case, either we have from the previous step and the induction assumption that either:
>- $D[u] = D[v] = \infty$ which results in $D[v] = \infty$ at the end of this step.
>- One of these distances if finite. In either case, if $D[v]$ remains unchanged, it must be finite and so according to the inductive step, it is the weight of a path from the source to $v$. If not, then the path from $s$ to $u$ and then hopping to $v$ has been chosen as the best path now, and thus corresponds with a path from the source to the node $v$.

And finally:

>[!NOTE] Lemma 2
>After $i$ repetitions of the loop, $D[u]$ is at most, the length of the shortest path from the source to $u$ with at most $i$ edges, if such a path exists. It will be infinity if not.

This is the real meat of the proof.

>[!IMPORTANT]- Proof
>The base case is the same as the previous lemma.
>
>For the inductive step, assume the lemma holds until the end of loop $i$. Using the previous lemma, either the distance to a node $v$ is infinite, or it is not and thus corresponds with some path from the source to $v$. 
>
>Consider one of the shortest paths from the source to node $v$ with at most $i+1$ edges (there can be multiple of these paths, just pick one). Call this path $P$.
>
>Now, $P$ ends with $v$ of course, and the previous node on the path is one of it's neighbors, let it be the node $u$ and weight of the edge $(u, v)$ be $w$. By Bellman's principle, the path from the source to $u$ is the shortest path from the source to $u$ with $i$ edges. Using the inductive assumption, we have that $D[u]$ equals at most to the weight of this path. 
>
>It is obvious that $D[u] + w$ is at most, the weight of the path $P$. Since we take the minimum of $D[v]$ and $D[u] + w$ and assign it to $D[v]$, it means that at the end of loop $i+1$, we have that $D[v]$ is at most the weight of $P$ and so the lemma is proven.

So according to this step, if loop $|V| - 1$ times, we would have found the shortest path of length at most $|V| - 1$ from the source to every node on the graph. Afterwards there is just no more paths left to consider and so we are done.

There is however a problem though, if negative cycles exists, paths can be improved by doing more loops! This is actually good, since we can use it to see if there are negative cycles in the first place. Here is how:

>[!NOTE] Theorem
>No improvements to the path lengths can be made after loop number $|V|$ if and only if there are no negative cycles in the graph.

>[!IMPORTANT]- Proof
>**Assume no improvement are made after the final loop**. Take any cycle in the graph like $\mathcal{C}$ and label the nodes from 0 to $|\mathcal{C}| = m$. So we can address these nodes as $\mathcal{C}[i]$ for $0 \leq i < m$. 
>
>Now, since no improvements were made, we have for all these nodes that:
>$$
>D[\mathcal{C}[i]] \leq D[u] + w(u, \mathcal{C}[i])
>$$
>for all $i$s and any neighbor of that node $u$. Pick the neighbor that follows the node in the cycle (there has to be one, since this node was part of the cycle) and sum up all the inequalities for all nodes in the cycle, you will end up with:
>$$
>\sum_{i=0}^{i=m-1} D[\mathcal{C}[i]] \leq \sum_{i=0}^{i=m-1} D[\mathcal{C}[i+1 \; mod \; m]] + \sum_{i=0}^{i=m-1} w(\mathcal{C}[i], \mathcal{C}[i+1 \; mod \; m])
>$$
>The two sums over each node are essentially the same, just shift the index of the second one by 1. So they can be subtracted from both sides, the last term however is the weight of the cycle $w(\mathcal{C})$, so we have:
>$$
>	0 \leq w(\mathcal{C})
>$$
>So the cycle weight is non-negative.
>
>
>**Assume no negative cycles exist**. In this case, every shortest path can be modified to visit each node at most once (we discussed it when we said that the subgraph of all paths, creates a DAG or a tree), which means that each shortest path will visit each node at most once. This means that during loop $|V| - 1$ we have considered any node that can be part of the shortest path and so no improvement will be made on the distances after this step.

### Handling Negative Cycles

With the above proof, it is easy to detect whether or not a negative cycle exists. Just run the loop one more time in the end and check if the results change at any point. If they do, then somewhere a negative loop looms.

## Complexity

It is easy to see that the algorithm runs in $\mathcal{O}(|V|.|E|)$ time.

Improvements can be made to the algorithm. One improvement relies on the fact that if the algorithm makes a step, *without* any change to the resulting values, then the algorithm can exit immediately (simply because nothing will change from now on, no matter how much we are behind the $|V| - 1$ mark). This helps, but it does not change the worst case runtime.
