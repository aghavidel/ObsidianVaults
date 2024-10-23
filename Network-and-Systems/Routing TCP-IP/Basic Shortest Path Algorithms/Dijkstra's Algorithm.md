# Intro.

Dijkstra's algorithm (named after Edgar W. Dijkstra), similar to [[Bellman-Ford]] algorithm, is an algorithm for solving the SSSP problem.

It runs faster than Bellman-Ford, but it only considers graphs with non-negative weights, which limits it's application somewhat. Regardless, it still uses the same principles and is recommended over Bellman-Ford when we are sure that our graphs do not contain any negative weights.

## The Algorithm

Unlike with Bellman-Ford, Dijkstra's algorithm does not necessarily check all neighbors of a node during relaxation. The algorithm maintains a set of *unvisited* nodes in a data structure called $Q$. At each relaxation step, only neighbors already residing in $Q$ will be considered, and so the algorithm will finish faster.

So the relaxation step will be:

```python
def relaxation(v, E):
	for each neighbor u of v still in Q:
		relaxed_distance = G.weight(v, u) + D[v]
		if relaxed_distance < D[u]:
			D[u] <-- relaxed_distance
			P[u] <-- v
```

We know from our discussion on Bellman-Ford's algorithm that the set of all shortest paths from a single source, create a DAG or a tree depending on whether or not we are working with an undirected graph. With this in mind, the full algorithm of Dijkstra will be the following:

```python
==========================
	Dijkstra
==========================
'''
Given G := (V, E)
Given some s in V as source
'''
--------------------------

'''Initialize'''
global D := list()
global P := list()
global Q := DS()
for u in V:
	D[u] <-- inf
	P[u] <-- None
	Q.add(u)
D[s] <-- 0

'''Loop'''
while Q is not empty:
	v <-- vertex in Q with minimum distance
	Q.remove(v)
	relaxation(v, E)

return D, P
```

The data structure `DS` should support:
- Key removal
- Key addition
- Minimum value extraction

Dijkstra's own version of the algorithm used a priority queue which prioritizes on distance from the source.

## Proof of Correctness

Proof is very much like Bellman-Ford's proof of correctness. The main difference here is that it's hard to say how many neighbors we are actually checking during each loop. 

That said, an invariant can still be created using the same techniques:

>[!NOTE] Lemma 1
>For each node $v$, at each step, the value $D[v]$ signifies the shortest path from the source to $v$, using only the nodes that are not in $Q$.
>If such a path does not exist, then the value will be infinity.

Proof is simple with induction on the number of nodes that have been popped from $Q$ (i.e. the number of nodes already visited).

>[!IMPORTANT] Proof
>The base case of only one visited node corresponds to the trivial case of starting from the source.
>
>For the inductive step, assume the lemma holds for $n$ nodes, and we just finished visiting the $n+1$ node, call that node $v$. 
>
>$v$ must be the unvisited node with minimum distance from the source and must be a neighbor of a node $u$ which was previously visited. Using the induction hypothesis, the distance from the source to $u$ equals the shortest path from the source to $u$ using the currently visited nodes. Assume that the path from the source to $v$ that passes through $u$ is *not* the shortest path.

