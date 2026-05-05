# Architecture

## Overview

The explorer is a two-part application. The frontend gives the learner an interactive workbench. The backend owns the planning algorithms and returns a trace that can be visualized one frame at a time.

## Backend

The backend is a FastAPI application in `backend/app`.

Main components:

- `grid.py`: parses and validates grid worlds, defines actions, transitions, and inverse transitions
- `models.py`: Pydantic request and response models shared by the API
- `search.py`: forward, backward, and bidirectional grid search implementations
- `weighted_graph.py`: Dijkstra, A*, forward value iteration, and backward value iteration over directed weighted graphs
- `code_judge.py`: live Python3 code execution and correctness evaluation
- `main.py`: FastAPI routes and CORS configuration

The main endpoint is:

```text
POST /api/chapter2/search/trace
```

The request includes an algorithm name and a grid. The response includes:

- status: found or not found
- plan: executable forward action sequence
- trace: ordered frames for visualization, including visited sets, frontiers, and search-tree edges
- stats: expanded count, visited count, maximum frontier size, path length, trace length

For weighted graph algorithms, the same endpoint accepts a `graph` object instead of a grid. A weighted graph contains `nodes`, directed `edges`, `start`, and `goal`. Each node may include an `(x, y)` position and a heuristic value for A*. The response includes the graph, `graph_path`, total cost, relaxation count, sweep count, and graph trace frames with node labels, priority queue snapshots, active relaxation edges, parent edges, and policy edges.

The live-code endpoints are:

```text
GET  /api/chapter2/code/default/{algorithm}
POST /api/chapter2/code/evaluate
POST /api/chapter2/code/visualize
```

The default-code endpoint returns the editable Python3 implementation for the selected algorithm. The visualization endpoint executes the submitted code, validates the current-grid action sequence, and returns a trace for the frontend. Default implementations return a dictionary with `actions` and `trace`; the trace records BFS frontier snapshots and parent-discovery edges. The runner still accepts a plain action list for learner code, but action-only submissions can only produce a path visualization. The evaluation endpoint runs the same submitted code against the current grid plus fixed judge cases.

The current judge is appropriate for local educational use. It is not a security boundary for arbitrary public users.

## Frontend

The frontend is a Vite React TypeScript app in `frontend/src`.

Main components:

- `App.tsx`: application state and layout
- `AlgorithmSelector.tsx`: Chapter 2 algorithm selection
- `GridEditor.tsx`: editable grid input
- `GraphEditor.tsx`: editable weighted graph JSON input
- `GridVisualization.tsx`: SVG visualization of trace frames
- `WeightedGraphVisualization.tsx`: SVG node-link visualization for weighted graph traces
- `TraceControls.tsx`: run, reset, step, and play controls
- `InspectorPanel.tsx`: explanation, complexity, stats, pseudocode, and trace messages
- `LiveCodeEditor.tsx`: Python3 editor, light/dark mode, exact-code evaluation, and judge results
- `LatexBlock.tsx`: KaTeX rendering for pseudocode

The visualization is SVG-based because the state spaces are small, inspectable teaching examples. Grid algorithms use a cell-based SVG. Weighted graph algorithms use a node-link SVG with edge costs, current relaxation, open/settled node states, and per-node values such as `g`, `h`, `f`, or `V`.

## Data Flow

1. The learner edits the grid, selects an algorithm, and edits the Python3 code.
2. The frontend validates simple grid shape errors.
3. The frontend posts the grid, algorithm, and exact editor code to the backend.
4. The backend executes the submitted code with a timeout and validates the returned action sequence.
5. The backend returns submitted trace frames when provided, or a path-only trace when the submitted code returns only actions.
6. The frontend renders any trace frame without rerunning the algorithm.

Weighted graph algorithms currently use the direct trace endpoint instead of the live-code judge. This keeps the graph trace contract stable while the editor-focused workflow remains available for the grid algorithms.

For live coding:

1. The frontend loads the default Python3 code for the selected algorithm.
2. The learner edits the code and submits it through `Run Code`.
3. The backend runs that exact code in the judge process.
4. The judge checks whether the returned actions are valid, shortest for the unweighted cases, and correct on no-path cases.
5. The frontend shows per-case pass/fail feedback.

## Trace Model

Each `TraceFrame` records the state of one expansion or update step. Forward search stores `visited`, `frontier`, and `forward_tree_edges`. Backward search stores `backward_visited`, `backward_frontier`, and `backward_tree_edges`. Bidirectional search fills both sides. The tree edges come from parent-pointer discoveries, so they show the explored BFS tree, while `plan_prefix` highlights the final path after a solution is reconstructed.

Weighted graph frames add graph-specific fields:

- `current_node`, `frontier_nodes`, `settled_nodes`, and `updated_nodes`
- `node_labels` for `g`, `h`, `f`, or `V`
- `active_edge` and `relaxation` for the Bellman or shortest-path update being inspected
- `priority_queue` for Dijkstra and A*
- `parent_edges` for shortest-path trees and `policy_edges` for value iteration policies

## Development Workflow

When adding a new algorithm, implement it first in the backend with tests. Then add the frontend content, controls, and visualization states needed to teach it.

For a new Chapter 2 algorithm, add:

- backend implementation and response trace
- backend correctness tests and trace invariant tests
- frontend algorithm metadata and pseudocode
- UI tests for selection, controls, and rendered stats
- live-code default implementation and judge cases when learners should edit it
- documentation notes in `docs/chapter-2-discrete-planning.md`

For weighted graph algorithms, also add a graph trace invariant test that checks final path cost, relaxation updates, and the relevant node labels.

## Deployment Shape

The default production target is Vercel:

- `frontend/` is built into static files served from Vercel's CDN.
- `api/index.py` exposes the existing FastAPI app as a Vercel Python Function.
- `vercel.json` rewrites `/api/*` to the Python Function and keeps the React single-page app fallback at `/index.html`.
