# Graphs

> **Tier:** T1 · **Related:** [Trees and BSTs](07-trees-bst.md), [Heaps](08-heaps.md), [Recursion and backtracking](05-recursion-backtracking.md), [STL for DSA](../03-languages/c-cpp/04-stl-for-dsa.md), [Background processing patterns](../13-architecture-and-practices/06-background-processing-patterns.md) · **Language:** C++17

## Quick revision
- A graph is a set of vertices (V) joined by edges (E); edges can be directed or undirected, weighted or unweighted.
- **Adjacency list** (`vector<vector<int>> adj(n)`): O(V + E) space, the default. **Matrix**: O(V^2) space, O(1) edge lookup, for dense graphs. **Edge list**: for Kruskal and Bellman-Ford.
- **BFS** uses a queue, explores in rings of distance, and finds shortest paths in unweighted graphs. Mark visited when you enqueue.
- **DFS** uses recursion or a stack and goes deep first: components, cycle detection, topological order, backtracking.
- BFS and DFS are O(V + E) with adjacency lists.
- A grid is a graph: cells are vertices, with up to four neighbors from direction arrays.
- Components: start a BFS/DFS from every unvisited vertex and count the starts.
- Undirected cycle: DFS meets a visited vertex that is not its parent, or union-find sees both endpoints already connected.
- Directed cycle: DFS with three colors; an edge into a gray (in-progress) vertex is a cycle.
- **Topological sort** (DAGs only): Kahn's in-degree queue or reverse DFS postorder, O(V + E).
- Shortest paths: BFS (unweighted), **Dijkstra** (non-negative weights, O((V + E) log V)), **Bellman-Ford** (negative weights allowed, O(V * E), detects negative cycles).
- Dijkstra breaks with negative edges: a vertex it already settled could later get a cheaper path.
- **Union-find** with path compression and union by rank/size: nearly O(1) amortized per operation.
- **MST:** Kruskal (sort edges + union-find) or Prim (min-heap).
- Signals: networks, connections, prerequisites, islands, "minimum steps/moves", dependencies, grouping.

## 1. Concept from scratch

### 1.1 Vocabulary
| Term | Meaning |
|---|---|
| Directed / undirected | edges have a direction (u -> v) or work both ways |
| Weighted | each edge carries a cost, distance or time |
| Degree | number of edges at a vertex (in-degree and out-degree for directed graphs) |
| Path, cycle | a sequence of edges; a path that returns to its start |
| Connected component | a maximal set of vertices that can all reach each other (undirected) |
| DAG | directed acyclic graph: directed, with no cycles |
| Sparse / dense | E close to V versus E close to V^2 |
| Tree | a connected, acyclic undirected graph with exactly V - 1 edges |

### 1.2 Representations
| | Adjacency list | Adjacency matrix | Edge list |
|---|---|---|---|
| Space | O(V + E) | O(V^2) | O(E) |
| Is there an edge u-v? | O(degree of u) | O(1) | O(E) |
| Visit the neighbors of u | O(degree of u) | O(V) | O(E) |
| Best for | most problems, sparse graphs | dense graphs, Floyd-Warshall | Kruskal, Bellman-Ford |

To build a list from LeetCode-style input `edges[i] = {u, v}`, push `v` into `adj[u]`, and also `u` into `adj[v]` when the graph is undirected. Weighted graphs store pairs: `vector<vector<pair<int, int>>> adj(n)` holding `(neighbor, weight)`.

### 1.3 BFS and DFS
- **BFS** visits vertices in order of their distance (in edges) from the source, using a FIFO queue. The first time it discovers a vertex, it has found a shortest path to it. Mark a vertex visited when you push it; marking on pop lets the same vertex enter the queue many times.
- **DFS** follows one path as deep as it can, then backtracks. It is the natural fit for "explore everything reachable", counting components, detecting cycles, topological sorting and path-existence questions.

| Use BFS when | Use DFS when |
|---|---|
| you need the fewest edges (shortest path in an unweighted graph or grid) | you need to explore everything, or detect cycles |
| you process level by level (spreading, "minutes until") | you need a topological order or subtree-style results |
| the graph is deep and recursion could overflow | the graph is wide and a BFS queue would be huge |

### 1.4 Grids as graphs
A cell `(r, c)` is a vertex whose neighbors are found with direction arrays `dr = {1, -1, 0, 0}` and `dc = {0, 0, 1, -1}` plus a bounds check. Mark visited either by overwriting the cell (when modifying the input is allowed) or in a separate `visited` matrix. With R x C cells, V = R * C and E is at most about 4V. **Multi-source BFS** starts with every source in the queue at distance 0, for example all rotten oranges at once.

### 1.5 Components and cycle detection
- **Components:** loop over all vertices; each unvisited vertex starts a new BFS/DFS and a new component. Union-find gives the same answer incrementally as edges arrive.
- **Undirected cycle:** in DFS, reaching an already visited neighbor that is not the vertex you came from means a cycle. With union-find, an edge whose endpoints already have the same root closes a cycle.
- **Directed cycle:** a plain `visited` flag is not enough, because reaching a finished vertex through a different branch is fine. Use three colors: white (unvisited), gray (on the current DFS path), black (finished). An edge to a gray vertex is a back edge, which means a cycle.

### 1.6 Topological sort
A topological order lists the vertices so that every edge u -> v has u before v. It exists exactly when the graph is a DAG.
- **Kahn's algorithm (BFS):** compute every in-degree, enqueue the vertices with in-degree 0, then repeatedly pop one, append it to the order, and decrement its neighbors' in-degrees, enqueueing any that reach 0. If the order ends up shorter than V, the leftover vertices sit on a cycle.
- **DFS:** append a vertex after all of its descendants are finished (postorder), then reverse the list.

Uses: build systems and CI pipelines, course prerequisites, data and job pipelines, package installation order, spreadsheet recalculation, and ordering database migrations or imports so parents exist before children.

### 1.7 Shortest paths
| Algorithm | Works on | Time | Idea |
|---|---|---|---|
| BFS | unweighted graphs | O(V + E) | the first discovery is the shortest |
| 0-1 BFS | weights 0 or 1 | O(V + E) | deque: 0-edges to the front, 1-edges to the back |
| Dijkstra | non-negative weights | O((V + E) log V) with a binary heap | always settle the closest unsettled vertex |
| Bellman-Ford | any weights, no reachable negative cycle | O(V * E) | relax every edge V - 1 times; any further improvement means a negative cycle |
| Floyd-Warshall | all pairs | O(V^3) | DP over intermediate vertices; fine for a few hundred vertices |

**Why Dijkstra needs non-negative weights.** When Dijkstra pops the unsettled vertex with the smallest tentative distance, it declares that distance final: every other route must pass through a vertex that is at least as far away, and edges can only add cost. A negative edge breaks the argument. Example: A -> B costs 1, A -> C costs 3, C -> B costs -3. Dijkstra settles B at 1, but the true distance is 0 via C. Use Bellman-Ford (or SPFA, or Johnson's algorithm for all pairs) when weights can be negative.

```cpp
#include <array>
#include <climits>
#include <vector>
using namespace std;

// Bellman-Ford over an edge list of (u, v, w). Returns false if a negative cycle is reachable.
bool bellmanFord(int n, const vector<array<int, 3>>& edges, int src, vector<long long>& dist) {
    const long long INF = LLONG_MAX / 4;
    dist.assign(n, INF);
    dist[src] = 0;
    for (int round = 0; round < n - 1; round++)          // a shortest path has at most n - 1 edges
        for (const auto& [u, v, w] : edges)
            if (dist[u] != INF && dist[u] + w < dist[v]) dist[v] = dist[u] + w;
    for (const auto& [u, v, w] : edges)
        if (dist[u] != INF && dist[u] + w < dist[v]) return false;   // still improving: negative cycle
    return true;
}
```

### 1.8 Union-find (disjoint set union)
Union-find keeps elements partitioned into disjoint sets. `find(x)` returns the set's representative (root), and `unite(a, b)` merges two sets. Two optimizations make it nearly constant time: **path compression** (during `find`, point every visited node straight at the root) and **union by rank or size** (attach the smaller tree under the larger). Together they give O(alpha(n)) amortized per operation, where alpha, the inverse Ackermann function, is at most 4 for any realistic n. Uses: dynamic connectivity, Kruskal's MST, cycle detection in undirected graphs, and grouping problems like Accounts Merge or Number of Provinces.

### 1.9 Going deeper: minimum spanning tree
An MST connects every vertex of a weighted undirected graph using V - 1 edges with the smallest possible total weight. **Kruskal:** sort edges by weight and add each edge whose endpoints are in different components (union-find), O(E log E). **Prim:** grow a tree from any vertex, always adding the cheapest edge that leaves it (min-heap), O(E log V). Real uses: laying cable or pipes at minimum cost, clustering. Practice: Min Cost to Connect All Points (LeetCode 1584).

### 1.10 Graphs in real systems
- **Job dependency DAGs:** CI pipelines, data pipelines and background-job workflows run a step only after its prerequisites succeed. A topological order is a valid execution order, independent steps can run in parallel, and a cycle is a configuration error to report.
- **Package dependency resolution:** Bundler, npm and pip build a graph from your Gemfile, package.json or requirements file, choose versions that satisfy every constraint (a constraint-solving problem layered on the graph), and install in dependency order; circular dependencies have to be detected and handled.
- Also: foreign keys between tables (import parents before children), "people you may know" (BFS to depth 2), routing and maps (Dijkstra or A*), permission inheritance through organization hierarchies in multi-tenant apps.

## 2. Core templates
```cpp
#include <climits>
#include <functional>
#include <numeric>
#include <queue>
#include <stack>
#include <utility>
#include <vector>
using namespace std;

// BFS: number of edges from src to every vertex, or -1 if unreachable.
vector<int> bfs(const vector<vector<int>>& adj, int src) {
    vector<int> dist(adj.size(), -1);
    queue<int> q;
    dist[src] = 0;
    q.push(src);
    while (!q.empty()) {
        int u = q.front(); q.pop();
        for (int v : adj[u]) {
            if (dist[v] != -1) continue;           // already discovered
            dist[v] = dist[u] + 1;                 // mark when enqueuing
            q.push(v);
        }
    }
    return dist;
}

// DFS, recursive and iterative: mark everything reachable.
void dfs(const vector<vector<int>>& adj, int u, vector<bool>& seen) {
    seen[u] = true;
    for (int v : adj[u])
        if (!seen[v]) dfs(adj, v, seen);
}

void dfsIterative(const vector<vector<int>>& adj, int src, vector<bool>& seen) {
    stack<int> st;
    st.push(src);
    while (!st.empty()) {
        int u = st.top(); st.pop();
        if (seen[u]) continue;                     // a vertex may be pushed more than once
        seen[u] = true;
        for (int v : adj[u])
            if (!seen[v]) st.push(v);
    }
}

// Kahn's algorithm: a topological order, or an empty vector if there is a cycle.
vector<int> topoSort(const vector<vector<int>>& adj) {
    int n = adj.size();
    vector<int> indeg(n, 0), order;
    for (int u = 0; u < n; u++)
        for (int v : adj[u]) indeg[v]++;
    queue<int> q;
    for (int u = 0; u < n; u++)
        if (indeg[u] == 0) q.push(u);
    while (!q.empty()) {
        int u = q.front(); q.pop();
        order.push_back(u);
        for (int v : adj[u])
            if (--indeg[v] == 0) q.push(v);
    }
    if ((int)order.size() < n) return {};          // the rest sit on a cycle
    return order;
}

// Dijkstra with lazy deletion; adj[u] holds (v, w) pairs with w >= 0.
vector<long long> dijkstra(const vector<vector<pair<int, int>>>& adj, int src) {
    vector<long long> dist(adj.size(), LLONG_MAX);
    priority_queue<pair<long long, int>, vector<pair<long long, int>>, greater<pair<long long, int>>> pq;
    dist[src] = 0;
    pq.push({0, src});
    while (!pq.empty()) {
        auto [d, u] = pq.top(); pq.pop();
        if (d > dist[u]) continue;                 // stale entry: u was settled more cheaply
        for (auto [v, w] : adj[u]) {
            if (d + w < dist[v]) {
                dist[v] = d + w;
                pq.push({dist[v], v});
            }
        }
    }
    return dist;
}

// Union-find with path compression and union by size.
struct DSU {
    vector<int> parent, sz;
    explicit DSU(int n) : parent(n), sz(n, 1) { iota(parent.begin(), parent.end(), 0); }
    int find(int x) { return parent[x] == x ? x : parent[x] = find(parent[x]); }
    bool unite(int a, int b) {
        a = find(a);
        b = find(b);
        if (a == b) return false;                  // already in the same set
        if (sz[a] < sz[b]) swap(a, b);
        parent[b] = a;                             // smaller tree goes under the larger
        sz[a] += sz[b];
        return true;
    }
};
```

## 3. Problems

### 3.1 Number of Islands (LeetCode 200, Medium)
**Problem (paraphrased):** in a grid of `'1'` (land) and `'0'` (water), count the islands, where an island is land cells joined horizontally or vertically.
**Approach:** without marking, you would re-explore the same island from each of its cells. Scan the grid; each unvisited land cell starts a flood fill (DFS or BFS) that sinks the whole island, and each start counts one island. Union-find over land cells is an alternative.
**Complexity:** O(R * C) time, O(R * C) worst-case recursion depth (use BFS or an explicit stack for very large grids).
```cpp
class Solution {
public:
    int numIslands(vector<vector<char>>& grid) {
        int islands = 0;
        for (int r = 0; r < (int)grid.size(); r++)
            for (int c = 0; c < (int)grid[0].size(); c++)
                if (grid[r][c] == '1') { islands++; sink(grid, r, c); }
        return islands;
    }
private:
    void sink(vector<vector<char>>& g, int r, int c) {      // flood fill: turn the island into water
        if (r < 0 || c < 0 || r >= (int)g.size() || c >= (int)g[0].size() || g[r][c] != '1') return;
        g[r][c] = '0';
        sink(g, r + 1, c); sink(g, r - 1, c); sink(g, r, c + 1); sink(g, r, c - 1);
    }
};
```
**Edge cases:** all water (0); all land (1); diagonal neighbors do not connect.

### 3.2 Max Area of Island (LeetCode 695, Medium)
**Problem (paraphrased):** in a 0/1 grid, return the size of the largest island (4-directional), or 0 if there is no land.
**Approach:** same flood fill as 3.1, but the DFS returns the number of cells it sinks.
**Complexity:** O(R * C) time, O(R * C) worst-case stack.
```cpp
class Solution {
public:
    int maxAreaOfIsland(vector<vector<int>>& grid) {
        int best = 0;
        for (int r = 0; r < (int)grid.size(); r++)
            for (int c = 0; c < (int)grid[0].size(); c++)
                best = max(best, area(grid, r, c));
        return best;
    }
private:
    int area(vector<vector<int>>& g, int r, int c) {
        if (r < 0 || c < 0 || r >= (int)g.size() || c >= (int)g[0].size() || g[r][c] != 1) return 0;
        g[r][c] = 0;                                         // mark visited
        return 1 + area(g, r + 1, c) + area(g, r - 1, c) + area(g, r, c + 1) + area(g, r, c - 1);
    }
};
```
**Edge cases:** no land; one big island covering the grid.

### 3.3 Clone Graph (LeetCode 133, Medium)
**Problem (paraphrased):** given one node of a connected undirected graph (each node has a value and a neighbor list), return a deep copy of the whole graph.
**Approach:** copying nodes naively loops forever on cycles or duplicates shared nodes. Keep a hash map from original to clone; BFS (or DFS) over the originals, creating each clone the first time a node is seen and wiring neighbor lists as you go.
**Complexity:** O(V + E) time, O(V) space.
```cpp
class Solution {
public:
    Node* cloneGraph(Node* node) {
        if (!node) return nullptr;
        unordered_map<Node*, Node*> cloneOf;
        cloneOf[node] = new Node(node->val);
        queue<Node*> q;
        q.push(node);
        while (!q.empty()) {
            Node* cur = q.front(); q.pop();
            for (Node* nb : cur->neighbors) {
                if (!cloneOf.count(nb)) {                    // first visit: clone and enqueue
                    cloneOf[nb] = new Node(nb->val);
                    q.push(nb);
                }
                cloneOf[cur]->neighbors.push_back(cloneOf[nb]);
            }
        }
        return cloneOf[node];
    }
};
```
**Edge cases:** null input; a single node with no neighbors; cycles.

### 3.4 Rotting Oranges (LeetCode 994, Medium)
**Problem (paraphrased):** in a grid of empty cells (0), fresh oranges (1) and rotten oranges (2), every minute each rotten orange rots its fresh 4-directional neighbors. Return the minutes until no fresh orange remains, or -1 if some never rot.
**Approach:** simulating minute by minute with full grid scans is O((R * C)^2). Multi-source BFS starts with every rotten orange in the queue; each BFS level is one minute, and a counter of fresh oranges tells you when to stop or whether some are unreachable.
**Complexity:** O(R * C) time and space.
```cpp
class Solution {
public:
    int orangesRotting(vector<vector<int>>& grid) {
        int rows = grid.size(), cols = grid[0].size(), fresh = 0;
        queue<pair<int, int>> q;
        for (int r = 0; r < rows; r++)
            for (int c = 0; c < cols; c++) {
                if (grid[r][c] == 2) q.push({r, c});        // every rotten orange starts at minute 0
                else if (grid[r][c] == 1) fresh++;
            }
        const int dr[4] = {1, -1, 0, 0}, dc[4] = {0, 0, 1, -1};
        int minutes = 0;
        while (!q.empty() && fresh > 0) {
            int size = q.size();
            for (int i = 0; i < size; i++) {                 // one level = one minute
                auto [r, c] = q.front(); q.pop();
                for (int d = 0; d < 4; d++) {
                    int nr = r + dr[d], nc = c + dc[d];
                    if (nr < 0 || nc < 0 || nr >= rows || nc >= cols || grid[nr][nc] != 1) continue;
                    grid[nr][nc] = 2;
                    fresh--;
                    q.push({nr, nc});
                }
            }
            minutes++;
        }
        return fresh == 0 ? minutes : -1;
    }
};
```
**Edge cases:** no fresh oranges at the start (0); a fresh orange walled off by empty cells (-1); no rotten oranges but some fresh (-1).

### 3.5 Pacific Atlantic Water Flow (LeetCode 417, Medium)
**Problem (paraphrased):** water flows from a cell to a 4-directional neighbor of equal or lower height. The Pacific touches the top and left edges, the Atlantic the bottom and right edges. List the cells from which water can reach both oceans.
**Approach:** running a search from every cell to see which oceans it reaches is O((R * C)^2). Reverse the flow: start from each ocean's coastline and walk uphill (to neighbors of equal or greater height), marking every cell that can drain into that ocean. Return the cells marked by both searches.
**Complexity:** O(R * C) time and space.
```cpp
class Solution {
public:
    vector<vector<int>> pacificAtlantic(vector<vector<int>>& heights) {
        int rows = heights.size(), cols = heights[0].size();
        vector<vector<bool>> pac(rows, vector<bool>(cols, false)), atl(rows, vector<bool>(cols, false));
        for (int r = 0; r < rows; r++) {
            climb(heights, pac, r, 0);              // left edge: Pacific
            climb(heights, atl, r, cols - 1);       // right edge: Atlantic
        }
        for (int c = 0; c < cols; c++) {
            climb(heights, pac, 0, c);              // top edge: Pacific
            climb(heights, atl, rows - 1, c);       // bottom edge: Atlantic
        }
        vector<vector<int>> result;
        for (int r = 0; r < rows; r++)
            for (int c = 0; c < cols; c++)
                if (pac[r][c] && atl[r][c]) result.push_back({r, c});
        return result;
    }
private:
    // Mark every cell that can drain to this ocean by walking uphill from the coast.
    void climb(const vector<vector<int>>& h, vector<vector<bool>>& seen, int r, int c) {
        if (seen[r][c]) return;
        seen[r][c] = true;
        const int dr[4] = {1, -1, 0, 0}, dc[4] = {0, 0, 1, -1};
        for (int d = 0; d < 4; d++) {
            int nr = r + dr[d], nc = c + dc[d];
            if (nr < 0 || nc < 0 || nr >= (int)h.size() || nc >= (int)h[0].size()) continue;
            if (h[nr][nc] >= h[r][c]) climb(h, seen, nr, nc);   // water could flow from there to here
        }
    }
};
```
**Edge cases:** a single cell (reaches both); a single row or column; flat plateaus.

### 3.6 Course Schedule (LeetCode 207, Medium)
**Problem (paraphrased):** there are n courses and pairs `[a, b]` meaning b must be taken before a. Decide whether all courses can be finished.
**Approach:** the courses form a directed graph; finishing is possible exactly when it has no cycle. Brute force could try every ordering (n!). Kahn's algorithm takes courses with no remaining prerequisites; if some course is never freed, it is stuck on a cycle.
**Complexity:** O(V + E) time and space.
```cpp
class Solution {
public:
    bool canFinish(int numCourses, vector<vector<int>>& prerequisites) {
        vector<vector<int>> adj(numCourses);
        vector<int> indegree(numCourses, 0);
        for (auto& p : prerequisites) {                     // p = {course, prereq}: edge prereq -> course
            adj[p[1]].push_back(p[0]);
            indegree[p[0]]++;
        }
        queue<int> q;
        for (int i = 0; i < numCourses; i++)
            if (indegree[i] == 0) q.push(i);
        int taken = 0;
        while (!q.empty()) {
            int u = q.front(); q.pop();
            taken++;
            for (int v : adj[u])
                if (--indegree[v] == 0) q.push(v);
        }
        return taken == numCourses;                         // leftovers are on a cycle
    }
};
```
**Edge cases:** no prerequisites (true); a self-loop `[a, a]` (false); several disconnected groups.

### 3.7 Course Schedule II (LeetCode 210, Medium)
**Problem (paraphrased):** same input as 3.6; return any valid order to take all courses, or an empty list if impossible.
**Approach:** Kahn's order works directly. Shown here is the DFS alternative with three colors: a gray neighbor means a cycle; finished courses are appended in postorder, and the reversed postorder is a topological order.
**Complexity:** O(V + E) time and space.
```cpp
class Solution {
public:
    vector<int> findOrder(int numCourses, vector<vector<int>>& prerequisites) {
        vector<vector<int>> adj(numCourses);
        for (auto& p : prerequisites) adj[p[1]].push_back(p[0]);
        vector<int> color(numCourses, 0);        // 0 = unvisited, 1 = on the current path, 2 = done
        vector<int> order;
        for (int i = 0; i < numCourses; i++)
            if (color[i] == 0 && !dfs(i, adj, color, order)) return {};
        reverse(order.begin(), order.end());     // reversed postorder is a topological order
        return order;
    }
private:
    bool dfs(int u, const vector<vector<int>>& adj, vector<int>& color, vector<int>& order) {
        color[u] = 1;
        for (int v : adj[u]) {
            if (color[v] == 1) return false;                              // back edge: cycle
            if (color[v] == 0 && !dfs(v, adj, color, order)) return false;
        }
        color[u] = 2;
        order.push_back(u);                      // everything that depends on u is already placed
        return true;
    }
};
```
**Edge cases:** a cycle anywhere (empty result); many valid orders (any is accepted).

### 3.8 Redundant Connection (LeetCode 684, Medium)
**Problem (paraphrased):** a tree on nodes 1..n had one extra edge added. Return an edge whose removal leaves a tree; if several qualify, return the one that appears last in the input.
**Approach:** brute force removes each edge (from the end) and checks connectivity with a DFS, O(E * (V + E)). Union-find processes edges in order: the first edge whose endpoints are already connected closes the only cycle, and it is the last cycle edge in input order.
**Complexity:** O(E * alpha(V)) time, O(V) space.
```cpp
class Solution {
    vector<int> parent, rnk;
    int find(int x) {
        if (parent[x] != x) parent[x] = find(parent[x]);   // path compression
        return parent[x];
    }
    bool unite(int a, int b) {
        a = find(a);
        b = find(b);
        if (a == b) return false;                          // already connected: this edge closes a cycle
        if (rnk[a] < rnk[b]) swap(a, b);
        parent[b] = a;                                     // union by rank
        if (rnk[a] == rnk[b]) rnk[a]++;
        return true;
    }
public:
    vector<int> findRedundantConnection(vector<vector<int>>& edges) {
        int n = edges.size();                              // a tree plus one edge: n edges on n nodes
        parent.resize(n + 1);
        rnk.assign(n + 1, 0);
        iota(parent.begin(), parent.end(), 0);             // every node starts as its own root
        for (auto& e : edges)
            if (!unite(e[0], e[1])) return e;
        return {};
    }
};
```
**Edge cases:** the redundant edge is the last edge; a cycle through node 1. The directed variant (LeetCode 685) is considerably trickier.

### 3.9 Network Delay Time (LeetCode 743, Medium)
**Problem (paraphrased):** a signal starts at node k of a directed graph with positive travel times. Return how long until every node has received it, or -1 if some node never does.
**Approach:** this asks for the largest shortest-path distance from k. Bellman-Ford is O(V * E) and Floyd-Warshall O(V^3); with non-negative weights Dijkstra is the right tool.
**Complexity:** O((V + E) log V) time, O(V + E) space.
```cpp
class Solution {
public:
    int networkDelayTime(vector<vector<int>>& times, int n, int k) {
        vector<vector<pair<int, int>>> adj(n + 1);              // u -> (v, w); nodes are 1-indexed
        for (auto& t : times) adj[t[0]].push_back({t[1], t[2]});
        vector<int> dist(n + 1, INT_MAX);
        priority_queue<pair<int, int>, vector<pair<int, int>>, greater<pair<int, int>>> pq;   // (dist, node)
        dist[k] = 0;
        pq.push({0, k});
        while (!pq.empty()) {
            auto [d, u] = pq.top(); pq.pop();
            if (d > dist[u]) continue;                          // stale entry
            for (auto [v, w] : adj[u]) {
                if (d + w < dist[v]) {
                    dist[v] = d + w;
                    pq.push({dist[v], v});
                }
            }
        }
        int worst = *max_element(dist.begin() + 1, dist.end());
        return worst == INT_MAX ? -1 : worst;
    }
};
```
**Edge cases:** an unreachable node (-1); a single node (0); parallel edges with different weights.

### 3.10 Word Ladder (LeetCode 127, Hard)
**Problem (paraphrased):** transform a begin word into an end word by changing one letter at a time, where every intermediate word must be in a given word list. Return the number of words in the shortest such sequence, or 0 if none exists.
**Approach:** words are vertices and one-letter differences are edges, and we need the fewest edges: BFS. Building the graph by comparing every pair of words is O(N^2 * L). Instead, generate neighbors on the fly by trying all 26 letters at each position and checking the word set; erase words from the set when enqueued so each is visited once.
**Complexity:** O(N * L * 26 * L) time (hashing each candidate costs O(L)), O(N * L) space.
```cpp
class Solution {
public:
    int ladderLength(string beginWord, string endWord, vector<string>& wordList) {
        unordered_set<string> unvisited(wordList.begin(), wordList.end());
        if (!unvisited.count(endWord)) return 0;
        queue<string> q;
        q.push(beginWord);
        unvisited.erase(beginWord);
        int steps = 1;                                   // the sequence length counts beginWord
        while (!q.empty()) {
            int size = q.size();
            for (int i = 0; i < size; i++) {
                string word = q.front(); q.pop();
                if (word == endWord) return steps;
                for (int pos = 0; pos < (int)word.size(); pos++) {
                    char original = word[pos];
                    for (char c = 'a'; c <= 'z'; c++) {
                        if (c == original) continue;
                        word[pos] = c;
                        if (unvisited.count(word)) {
                            unvisited.erase(word);       // mark visited when enqueuing
                            q.push(word);
                        }
                    }
                    word[pos] = original;                // restore before the next position
                }
            }
            steps++;
        }
        return 0;
    }
};
```
**Edge cases:** the end word missing from the list (0); begin and end one letter apart (2). Bidirectional BFS from both ends cuts the explored frontier considerably.

## Common pitfalls
- Marking vertices visited when popping instead of when pushing in BFS, which enqueues duplicates and can blow up time and memory.
- Using a single `visited` flag for directed cycle detection; you need the gray/black distinction.
- Getting edge direction backwards in prerequisite problems (`[a, b]` means b before a).
- Recursive DFS on large grids or long chains overflowing the stack; switch to BFS or an explicit stack.
- Running Dijkstra with negative weights, or forgetting the stale-entry check (`d > dist[u]`) so old entries do extra work.
- `INT_MAX + w` overflow when relaxing from an unreached vertex; skip unreached vertices or use `long long` with a safe infinity.
- Forgetting disconnected parts: loop over every vertex as a possible start.
- Mutating the input grid when the interviewer expects it unchanged; ask first or use a visited matrix.

## How to talk about this in an interview
- First say what the vertices and edges are ("each cell is a vertex, edges go to the four neighbors"; "each course is a vertex, prerequisites are directed edges"), then which question you are asking (reachability, shortest path, ordering, connectivity). The algorithm follows from that.
- Justify the choice: "Unweighted and I need the fewest steps, so BFS"; "weights are positive, so Dijkstra"; "I need an order that respects dependencies, so topological sort." Explaining this mapping was core to teaching graph labs, so present it the way you would to a student.
- Real-world links: background jobs that depend on other jobs form a DAG, package managers resolve dependency graphs, and imports must insert parent records before children because of foreign keys. See [background processing patterns](../13-architecture-and-practices/06-background-processing-patterns.md).
- > **Fill in:** a dependency-ordering problem you met at work (sync jobs that must run in sequence, data imports with foreign keys, migration order) and how it was handled.

## Interview questions

**Q1. BFS versus DFS: how do they differ, and when do you use each?**
BFS uses a queue and explores vertices in order of distance from the start, so it finds shortest paths in unweighted graphs and suits level-by-level processing; its memory grows with the frontier's width. DFS uses a stack or recursion and goes deep first, which suits exhaustive exploration, cycle detection, topological sorting and backtracking; its memory grows with depth. Both are O(V + E).
*Follow-up:* Can DFS find the shortest path in an unweighted graph? Not directly; the first path it finds can be long. Use BFS.

**Q2. Adjacency list or adjacency matrix: how do you choose?**
Lists use O(V + E) memory and iterate neighbors in O(degree), which is ideal for sparse graphs (most real graphs). A matrix uses O(V^2) memory but answers "is there an edge?" in O(1), which suits dense graphs and algorithms like Floyd-Warshall.
*Follow-up:* What about a graph with a million vertices and a few million edges? Definitely a list; a matrix would need 10^12 cells.

**Q3. What is topological sorting used for?**
Ordering tasks with dependencies so that each runs after its prerequisites: build systems and CI pipelines, course planning, job and data pipelines, package installation, spreadsheet recalculation, and database migration or import order. It exists only for DAGs, and Kahn's algorithm doubles as cycle detection.
*Follow-up:* How do you find steps that can run in parallel? Process Kahn's queue level by level; each level's vertices have no dependencies among themselves.

**Q4. Why does Dijkstra fail with negative edges, and what do you use instead?**
Dijkstra finalizes the closest unsettled vertex, assuming no later path can be cheaper because edges only add cost. A negative edge can make a path through a farther vertex cheaper after a vertex is settled, so results can be wrong. Use Bellman-Ford (O(V * E), and it detects negative cycles) or Johnson's algorithm for all pairs.
*Follow-up:* What does a negative cycle mean for shortest paths? They are undefined, because you could loop forever to lower the cost.

**Q5. How do you detect a cycle in a directed versus an undirected graph?**
Undirected: DFS that meets an already visited vertex other than its parent, or union-find that finds both endpoints of an edge in the same set. Directed: DFS with three colors, where an edge to a gray (in-progress) vertex is a back edge; or Kahn's algorithm leaving vertices unprocessed.
*Follow-up:* Why doesn't the undirected rule work for directed graphs? Reaching a finished vertex through another branch is not a cycle in a directed graph.

**Q6. What is union-find, and what do path compression and union by rank give you?**
A structure that tracks disjoint sets with `find` (which set is this in?) and `unite` (merge two sets). Path compression flattens trees during `find`, and union by rank or size keeps trees shallow; together they make operations nearly O(1) amortized (inverse Ackermann). Uses: connectivity as edges arrive, Kruskal's MST, cycle detection, grouping accounts.
*Follow-up:* What can't it do well? Split sets or delete edges; it only merges.

**Q7. Why mark vertices visited when you enqueue them in BFS?**
Otherwise a vertex can be enqueued once for every neighbor that discovers it before it is processed, which wastes time and memory (badly on dense graphs and grids) and can break level counting. Marking on enqueue guarantees each vertex enters the queue once.
*Follow-up:* Is it the same for Dijkstra? No; Dijkstra may push a vertex several times with improving distances, so it checks for stale entries when popping instead.

**Q8. What is the complexity of BFS, DFS and Dijkstra?**
BFS and DFS are O(V + E) time with adjacency lists (O(V^2) with a matrix) and O(V) extra space. Dijkstra with a binary heap is O((V + E) log V); with lazy deletion the heap can hold O(E) entries, which is still O(E log V) asymptotically.
*Follow-up:* For a grid with R rows and C columns? V = R * C and E is about 4V, so BFS is O(R * C).

**Q9. Kruskal versus Prim for a minimum spanning tree?**
Kruskal sorts all edges and adds the cheapest edge that joins two components (union-find): O(E log E), simple, good for sparse graphs given as edge lists. Prim grows one tree from a start vertex using a min-heap of crossing edges: O(E log V), convenient with adjacency lists and dense graphs.
*Follow-up:* Is the MST unique? It is when all edge weights are distinct.
