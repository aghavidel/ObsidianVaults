This document will describe the implementation for NCFlow. The code in question is the NSDI submission that can be found [here]([netcontract/ncflow: Anonymized code for NCFlow, NSDI 2021 Spring submission](https://github.com/netcontract/ncflow)).

# Edge-Based Formulation

We first start with the edge based formulation. In this formulation, flows of any particular commodity are assigned per edge. We denote the flow for commodity $k$ on edge $(u, v)$ by $f_k(u, v)$. The demand for commodity $k$ is denoted by $d_k$, and the source and destination expected for commodity $k$ is denoted by $s_k$ and $t_k$.

So the edge formulation would have the following feasibility conditions:
$$
\begin{aligned}
	&\sum_{v}^{V} f_k(s_k, v) \leq d_k &&\quad 1 \leq k \leq K\\
	&\sum_{w \in V} f_k(s_k, w) - \sum_{w \in V} f_k(w, t_k) = 0 &&\quad 1\leq k \leq K \\
	&\sum_{w \in V} f_k(w, s_k) = 0 &&\quad 1 \leq k \leq K\\
	&\sum_{w \in V} f_k(t_k, w) = 0 &&\quad 1 \leq k \leq K \\
	&\sum_{k}^{K} f_k(u, v) \leq c(u, v) &&\quad (u, v) \in E\\
	&\sum_{w \in V} f_k(u, w) - \sum_{w \in V} f_k(w, u) = 0 &&\quad u \in V / \{s_k, t_k\},\; 1\leq k \leq K \\
\end{aligned}
$$
The conditions are respectively:

1. Demand constraint. The source produces no more than the expected demand $d_k$ for each commodity.
2. Flow from source must equal flow into target.
3. Flow into source must be 0.
4. Flow out of target must be 0.
5. Total flow over any edge must respect the capacity of that edge (denoted by $c(.,.)$)
6. Transit nodes $w$ conserve ingress and egress flows.

The matrix version for demand constraints is much cleaner, but the above suffices as well.

As for the objectives, for now, we will consider maximum flow, so maximize $\sum_{k}^{K} \sum_{v}^{V} f_k(s_k, v)$.

Before we see how it is coded, some background on how the code is actually organized.

## LP Solver Implementation

The code uses Gurobi (of course it does!), and under `lib/lp_solver.py`, the main wrapper class around Gurobi is defined:

```python
class LpSolver:

    def __init__(
	    self, model, debug_fn=None, DEBUG=False,
	    VERBOSE=False, out=None, gurobi_out='')
	
	def _print(self, *args)
	
	def solve_lp(
		self, method=Method.CONCURRENT, bar_tol=None, 
		err_tol=None, numeric_focus=False)
```
This accepts a Gurobi model object and objective and adds them as properties. The main thing to note here though is the `solve_lp` method.

```python
    def solve_lp(...):

        model = self._model
        model.setParam('Method', method.value)
        model.setParam('LogFile', self.gurobi_out)

        if numeric_focus:
            model.setParam('NumericFocus', 1)
		if bar_tol:
			model.Params.BarConvTol = bar_tol
		if err_tol:
			model.Params.OptimalityTol = err_tol
			model.Params.FeasibilityTol = err_tol

        try:
			# ...
            model.optimize()
			# ...
            return model.objVal
        except GurobiError as e:
            self._print('Error code ' + str(e.errno) + ': ' + str(e))
        except AttributeError as e:
            self._print(str(e))
            self._print('Encountered an attribute error')
```

The only Gurobi attributes exposed are: 
- `NumericFocus`: Used to signal Gurobi to handle sensitive numeric operations more carefully. Ideally, you should try to formulate the problem such that this is not needed, thus it is set to automatic be default.
  Gurobi will issue a warning if numeric troubles arise ... 
- `BarConvTol`: For interior point method, when the difference between primal and dual objective is less than this value, we terminate.
- `OptimalityTol`: For anything other than interior point method, this sets the tolerance for the difference between dual and primal objective for declaring optimality.
- `FeasibilityTol`: This sets the maximum gap that constraints can have before they are considered to violate the problem formulation.

This is all we need to know about the LP solver used in NCFlow, so now, we can discuss formulations.

## LP Formulation

Under `algorithms/abstract_formulation.py`, we have the following template for formulations:

```python
class AbstractFormulation:
	def __init__(self, objective, DEBUG=False, VERBOSE=False, out=None)
	
	def _print(self, *args)
	def _extract_inds_from_var_name(self, varName, var_group_name='f')
	def _create_sol_dict(self, sol_dict_def, commodity_list)
	def _construct_lp(self, fixed_total_flows=[])
	def _save_pkl(self, obj, fname)
	def _save_txt(self, obj, fname)
	
	def solve(self, problem, fixed_total_flows=[], **args)
	def solve_warm_start(self, problem)
	def extract_sol_as_dict(self)
	def extract_sol_as_mat(self)
```

In the above, the private methods can be pretty much ignored for now, with the exception of `_construct_lp`, which returns a `LpSolver` instance that we can use for Gurobi. The method `solve`, calls `optimize` on the model invokes Gurobi to take care of the rest.

For Simplex and Dual-Simplex, using warm start would be of tremendous help, since it allows Gurobi to use a near optimal base for most of the iterations (we'll discuss more on this once we look at path based formulation).

The results can be queried using the two `extract_sol_as_ ...` methods.

## Inputs

### Traffic Matrices

Under `lib/traffic_matrix.py`, we have the following:

```python
class TrafficMatrix:
	def __init__(self, problem, tm, seed, scale_factor)
	
	@classmethod
	def from_file(cls, fname)
	
	def copy(self)
	def serialize(self, dir_path, fmt='pickle')
	def perturb_matrix(self, mean, stddev)
	def perturb_matrix_mult(self, mean, stdev, seed_prob_tm)
	def update_matrix(self, scale_factor, type, **kwargs)
	
	def _init_traffic_matrix(self)
	def _update(self, type, **kwargs)
```
The above is mostly an abstract class. A traffic matrix is just a square matrix with diagonal entries set to 0. `pertrub_ ...` methods are used to randomly change some entries in the matrix and `update_matrix` is used to update the matrix as a whole.

Different implementations of the matrix can be used, we focus on a few of them here ...

#### Uniform

We let $T_{ij} \;{\Huge \textasciitilde}\; U[0, M]$ :
```python
def _init_traffic_matrix(self):
	np.random.seed(self.seed)
	num_nodes = len(self.problem.G.nodes)
	self._tm = np.random.rand(num_nodes, num_nodes) * self._max_demand
	self._tm = self._tm.astype(np.float32)
	np.fill_diagonal(self._tm, 0.0)
```
#### Exponential

We let $T_{ij} \;{\Huge \textasciitilde}\; \text{exp}(\beta \gamma^{d(i, j)})$, where $\beta$ and $\gamma$ are matrix parameters and $d(i, j)$ is the distance between nodes $i$ and $j$.
```python
def _init_traffic_matrix(self):
	G = self.problem.G
	np.random.seed(self.seed)
	num_nodes = len(G.nodes)
	distances = np.zeros((num_nodes, num_nodes), dtype=np.int)
	dist_iter = nx.shortest_path_length(G)
	
	for src, dist_dict in dist_iter:
		for target, dist in dist_dict.items():
			distances[src, target] = dist
			
	self._tm = np.array([
		[np.random.exponential(
			self._beta * (self._decay**dist))\
				for dist in row
		] for row in distances
	], dtype=np.float32)
	
	np.fill_diagonal(self._tm, 0.0)
	self._tm *= self._const_factor
```
#### Poisson

We let $T_{ij} \;{\Huge \textasciitilde}\; \text{poisson}(\lambda \gamma^{d(i, j)})$, where $\lambda$ and $\gamma$ are matrix parameters and $d(i, j)$ is the distance between nodes $i$ and $j$.
```python
def _init_traffic_matrix(self):
	G = self.problem.G
	np.random.seed(self.seed)
	num_nodes = len(G.nodes)
	distances = np.zeros((num_nodes, num_nodes), dtype=np.int)
	dist_iter = nx.shortest_path_length(G)
	for src, dist_dict in dist_iter:
		for target, dist in dist_dict.items():
			distances[src, target] = dist
	
	self._tm = np.array([
		[np.random.poisson(
			self._lam * (self._decay**dist))\
				for dist in row
		] for row in distances
	], dtype=np.float32)
	
	np.fill_diagonal(self._tm, 0.0)
	self._tm *= self._const_factor
```

#### Gravity

```python
def _init_traffic_matrix(self):
	G = self.problem.G
	np.random.seed(self.seed)
	num_nodes = len(G.nodes)
	self._tm = np.zeros((num_nodes, num_nodes), dtype=np.float32)
	sccs = nx.strongly_connected_components(G)
	
	for scc in sccs:
		in_cap_sum, out_cap_sum = defaultdict(float), defaultdict(float)
		for u in scc:
			for v in G.predecessors(u):
				in_cap_sum[u] += G[v][u]['capacity']
			for v in G.successors(u):
				out_cap_sum[u] += G[u][v]['capacity']
				
		in_cap_sum, out_cap_sum = dict(in_cap_sum), dict(out_cap_sum)
		in_total_cap = sum(in_cap_sum.values())
		out_total_cap = sum(out_cap_sum.values())

		for u in scc:
			norm_u = out_cap_sum[u] / out_total_cap
			for v in scc:
				if u == v:
					continue
				frac = norm_u * in_cap_sum[v] / \
					(in_total_cap - in_cap_sum[u])
				
				if self.random:
					self._tm[u, v] = max(
						np.random.normal(frac, frac / 4), 0.0)
				else:
					self._tm[u, v] = frac

	self._tm *= self._total_demand
```

### Problem Description

Under `lib/problem.py`, we have the following:

```python
class Problem:
    def __init__(
	    self, G, traffic_matrix=None, model='gravity', 
	    seed=0, scale_factor=1.0, **kwargs)
	def __setattr__(self, name, value)
	
	@staticmethod
	def _read_graph_json(fname)
	@staticmethod
	def _write_graph_json(G, fname)
	@staticmethod
	def _read_graph_graphml(fname)
	@staticmethod
	def _read_graph_dot(fname, old_way=True)
	
	@classmethod
	def from_file(cls, topology_fname, traffic_matrix_fname, old_way=True)
	@classmethod
	def fixed_traffic_matrix_problem(cls, G, traffic_matrix, seed=0)
	
	def copy(self)
	def print_stats(self)
	def intra_and_inter_demands(self, partitioner)
	def new_capacities(
		self, *, min_cap, max_cap, fixed_caps=[], same_both_ways=True)
	
	def _change_capacities(
		self, *, min_cap, max_cap, fixed_caps=[], same_both_ways=True)
	def _invalidate_commodity_lists(self)
```

The problem input is defined by an input graph, a traffic matrix (or one that is automatically generated, defaulting to a gravity model), a randomizer seed and a scale factor.
## Formulation

Under `algorithms/edge_formulation.py`, we have the following:

```python
class EdgeFormulation(AbstractFormulation):
	@classmethod
	def new_max_flow(cls, out=None):
		return cls(
			objective=Objective.MAX_FLOW, DEBUG=True, VERBOSE=True, out=out)
	
	def _construct_lp(self, fixed_total_flows=[])
```

The method `_construct_lp` is our bread and butter, here it is (slightly shortened ...):

```python
def _construct_lp(self, fixed_total_flows=[]):
	m = Model("max-flow: edge-formulation")

	G = self.problem.G                         # our graph
	COMMODITIES = self.problem.commodity_list  # list of commodities
	M = len(G.edges)                           # number of edges
	K = len(COMMODITIES)                       # number of commodity flows

	"""
	A matrix of M by K will be our flows.
	Each row of this would give the set of flows on a particular
	edge, indexed per commodity.
	"""
	self.vars = m.addVars(M, K, vtype=GRB.CONTINUOUS, lb=0.0, name='f')
	self.edges_list = list(G.edges.data('capacity'))

	"""
	Edge capacity constraints, sum up all entries in each row and
	assert that they must be smaller than the capacity.
	"""
	m.addConstrs(
		self.vars.sum(e, '*') <= c_e \
			for e, (_, _, c_e) in \
				enumerate(G.edges.data('capacity'))
	)
	
	"""
	Demand constraints at src/target and flow conservation constraints
	"""
	for k, (_, (src, target, d_k)) in enumerate(COMMODITIES):
		flow_out = defaultdict(list)
		flow_in = defaultdict(list)
		for e, edge in enumerate(G.edges()):
			flow_out[edge[0]].append(self.vars[e, k])
			flow_in[edge[1]].append(self.vars[e, k])

		# Flow out from source is no more than demand
		m.addConstr(quicksum(flow_out[src]) <= d_k)
		# Flow conservation from source to destination
		m.addConstr(quicksum(flow_out[src]) - quicksum(flow_in[target]) == 0)
		# Nothing goes into the source or out of the destination
		m.addConstr(quicksum(flow_in[src]) + quicksum(flow_out[target]) == 0)
		# Flow conservation in transit nodes
		for n in G.nodes():
			if n != src and n != target:
				m.addConstr(
					quicksum(flow_out[n]) - quicksum(flow_in[n]) == 0
				)

	# Set objective
	if self._objective == Objective.MAX_FLOW:
		obj = quicksum([
			self.vars[e, k]
			for k, (_, (s_k, _, d_k)) in \
				enumerate(COMMODITIES) \
					for e, (src, _) in \
						enumerate(G.edges())
			if src == s_k
		])
	
	elif self._objective == Objective.MAX_MIN_FAIRNESS:
		# LATER ...
	
	m.setObjective(obj, GRB.MAXIMIZE)

	return LpSolver(m, self.debug_fn, self.DEBUG, self.VERBOSE, self.out)
```
