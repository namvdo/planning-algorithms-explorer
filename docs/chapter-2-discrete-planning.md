# Chapter 2: Discrete Planning Notes

Chapter 2 of LaValle's _Planning Algorithms_ introduces planning in discrete state spaces. This is the right starting point because the state space can be represented as a graph, and the main question is clear: how can an algorithm find a sequence of actions from an initial state to a goal state?

## Core Model

A discrete planning problem has:

- a state space `S`
- an initial state `x_init`
- a goal state or goal test
- an action set
- a transition function that maps a state and action to a next state

For the first version of this project, a grid world is used as the state space. Each free cell is a state. The actions are `up`, `down`, `left`, and `right`. Walls remove states from the graph.

This is simple, but it is still useful. It makes the search frontier, visited set, parent pointers, and final plan visible.

The visualization now separates these ideas. Colored cells show the states that have been visited or are still in the frontier. Thin tree edges show parent-pointer discoveries. The final plan is drawn as a stronger path on top of the explored tree. This makes it easier to see why breadth-first search may inspect many states that are not part of the final solution.

## Forward Search

Forward search starts at the initial state and expands reachable states. With a FIFO queue, it becomes breadth-first search. On an unweighted graph, breadth-first search returns a shortest path in number of actions.

Important learning points:

- the frontier contains states discovered but not expanded
- the visited set prevents repeated work
- parent pointers reconstruct the final plan
- the search tree is usually larger than the final path
- the worst case can grow quickly when the branching factor is high

## Backward Search

Backward search starts from the goal and expands predecessor states. It is useful when reverse actions are easy to compute and when the goal side of the problem is smaller than the initial side.

The returned plan must still be executable forward from the initial state. This is why the backend stores the forward action while expanding backward.

## Bidirectional Search

Bidirectional search grows two frontiers: one from the initial state and one from the goal. When the frontiers meet, the algorithm joins the two partial plans.

The usual intuition is that two searches of depth `d / 2` may be much smaller than one search of depth `d`. This advantage depends on the problem. Bidirectional search needs a useful way to expand backward and a cheap way to detect when the frontiers meet.

## Metrics To Show

The first MVP reports:

- expanded states
- visited states
- maximum frontier size
- path length
- trace length

These metrics are good for learning because they connect the visual behavior to computational cost. Later versions can add runtime, memory estimates, weighted costs, and comparisons across algorithms.

## Weighted Graph Algorithms

The weighted algorithms use a directed graph instead of the grid world. This is a better visual model for Dijkstra, A\*, and value iteration because the edge cost is the main object being reasoned about. A weighted grid can still be derived later by treating each cell as a node and each move as a weighted edge, but the node-link graph is clearer for the learning interface that we want to build.

The weighted graph view shows:

- edge costs directly on the graph
- the current node being expanded or updated
- the active edge relaxation
- open/frontier nodes and settled nodes
- per-node labels such as `g`, `h`, `f`, or `V`
- parent edges for shortest-path reconstruction
- policy edges for backward value iteration

### Dijkstra

Dijkstra computes the optimal cost-to-come `g(x)` from the start to each node. The priority queue is ordered by the smallest known `g` value. When a node is settled, its `g` value is final. The main visual event is edge relaxation:

```text
g(x') = min(g(x'), g(x) + cost(x, x'))
```

The visualization should make it clear that Dijkstra expands outward by cost, not by number of edges.

### A\*

A\* also maintains `g(x)`, but the queue is ordered by:

```text
f(x) = g(x) + h(x)
```

The heuristic `h(x)` estimates the remaining cost to the goal. In the sample graph, each node has a heuristic value in the graph JSON. The important learning point is that A\* is still doing relaxation, but the priority queue is biased toward nodes that appear closer to the goal.

### Forward Value Iteration

Forward value iteration computes cost-to-come values from the start by repeated Bellman updates over incoming edges:

```text
g(x') = min over predecessors x of g(x) + cost(x, x')
```

The visualization shows sweeps and changed node values. This helps distinguish value iteration from queue-based graph search: it repeatedly improves a value function until no update changes the result.

### Backward Value Iteration

Backward value iteration anchors the goal at `V(goal) = 0` and computes cost-to-go:

```text
V(x) = min over successors x' of cost(x, x') + V(x')
```

The policy arrows show which successor currently gives the best cost-to-go. Once the values converge, following the policy from the start gives the optimal path.

## References

- Steven M. LaValle, _Planning Algorithms_, Chapter 2: Discrete Planning, Cambridge University Press, 2006.
- Official book page: https://lavalle.pl/planning/web.html
- Chapter 2 PDF: https://lavalle.pl/planning/ch2.pdf
- Cambridge University Press chapter page: https://www.cambridge.org/core/books/planning-algorithms/discrete-planning/D5B4A1A618C89DDB2E0D5C55A6060F30

## Live Coding Notes

The live editor uses Python3 because it matches the early Chapter 2 notes and is easy to read while studying algorithms. The default code is correct and executable. A learner can change it, run the visualization with that exact code, and run the judge to see whether the submitted code still returns valid plans.

The judge checks action sequences, not only whether a function returns something. This matters because a planning algorithm is correct only if its returned plan can actually be executed from the initial state to the goal.
