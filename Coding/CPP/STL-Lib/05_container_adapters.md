# 05 — Container Adapters

> `stack`, `queue`, `priority_queue`. Wrappers that restrict access patterns of an underlying container.

## Setup
```cpp
#include <bits/stdc++.h>
using namespace std;
```

---

## `std::stack` — LIFO

```cpp
stack<int> st;
st.push(1);
st.push(2);
st.push(3);
cout << st.top();    // => 3
st.pop();            // does NOT return value
cout << st.size();   // => 2
cout << st.empty();  // => 0 (false)
```

### Underlying container
Default is `deque<T>`. Override:
```cpp
stack<int, vector<int>> svec;     // vector-backed
```

### Use cases
- DFS (explicit stack)
- Parentheses / bracket matching
- Expression evaluation
- Monotonic stack (next greater element, histogram)

### Monotonic stack example: next greater element
```cpp
vector<int> nge(const vector<int>& a) {
    int n = a.size();
    vector<int> ans(n, -1);
    stack<int> st;   // stores indices, values decreasing top-to-bottom
    for (int i = 0; i < n; ++i) {
        while (!st.empty() && a[st.top()] < a[i]) {
            ans[st.top()] = a[i];
            st.pop();
        }
        st.push(i);
    }
    return ans;
}
```

---

## `std::queue` — FIFO

```cpp
queue<int> q;
q.push(1);
q.push(2);
q.push(3);
cout << q.front();   // => 1
cout << q.back();    // => 3
q.pop();             // removes front; doesn't return it
cout << q.size();    // => 2
```

### Use cases
- BFS
- Level-order tree traversal
- Producer/consumer scheduling

### BFS skeleton
```cpp
void bfs(const vector<vector<int>>& g, int src) {
    vector<int> dist(g.size(), -1);
    queue<int> q;
    q.push(src); dist[src] = 0;
    while (!q.empty()) {
        int u = q.front(); q.pop();
        for (int v : g[u]) if (dist[v] == -1) {
            dist[v] = dist[u] + 1;
            q.push(v);
        }
    }
}
```

---

## `std::priority_queue` — heap

### Max-heap (default)
```cpp
priority_queue<int> pq;
pq.push(3);
pq.push(1);
pq.push(4);
cout << pq.top();   // => 4
pq.pop();
cout << pq.top();   // => 3
```
**Complexity:** push O(log n), pop O(log n), top O(1).

### Min-heap
```cpp
priority_queue<int, vector<int>, greater<int>> mn;
mn.push(3); mn.push(1); mn.push(4);
cout << mn.top();   // => 1
```

### Pair ordering (lex by first, then second)
```cpp
priority_queue<pair<int,int>> pq;
pq.push({1, 100});
pq.push({2, 50});
pq.push({2, 90});
cout << pq.top().first << ',' << pq.top().second;   // => 2,90
```

### Custom struct ordering
```cpp
struct Edge { int to, cost; };
struct Cmp {
    bool operator()(const Edge& a, const Edge& b) const {
        return a.cost > b.cost;   // min-heap by cost
    }
};
priority_queue<Edge, vector<Edge>, Cmp> pq;
```

### Use cases
- Dijkstra's algorithm
- Top-K elements
- Median maintenance (two heaps)
- Greedy scheduling

### Dijkstra skeleton
```cpp
vector<long long> dijkstra(const vector<vector<pair<int,int>>>& g, int src) {
    vector<long long> dist(g.size(), LLONG_MAX);
    priority_queue<pair<long long,int>,
                   vector<pair<long long,int>>,
                   greater<>> pq;     // min-heap
    dist[src] = 0;
    pq.push({0, src});
    while (!pq.empty()) {
        auto [d, u] = pq.top(); pq.pop();
        if (d > dist[u]) continue;
        for (auto [v, w] : g[u]) {
            if (dist[u] + w < dist[v]) {
                dist[v] = dist[u] + w;
                pq.push({dist[v], v});
            }
        }
    }
    return dist;
}
```

**Gotcha:** `priority_queue` has no `decrease-key`. Re-insert with new priority and skip stale entries via the `d > dist[u]` check.

## See also
- [20 Comparators](20_comparators.md) for custom orderings in `priority_queue`
- [Heap algorithms (11)](11_algorithms_set_heap_minmax.md) if you need raw heap control on a vector
