# 23 — STL in Competitive Programming

> The subset of STL that wins contests, plus problem-pattern skeletons.

## Setup
```cpp
#include <bits/stdc++.h>
using namespace std;
```

---

## Essential containers

| Container | Why |
|---|---|
| `vector<T>` | the default array |
| `string` | char vector with goodies |
| `pair<A,B>`, `tuple<...>` | multi-return, multi-key sort |
| `set<T>` / `map<K,V>` | O(log n) ordered queries |
| `unordered_map<K,V>` | O(1) average lookup (watch for anti-hash, see [21](21_hashing.md)) |
| `priority_queue` | top-K, Dijkstra, greedy |
| `deque` | both-ends, sliding window |
| `stack`, `queue` | DFS, BFS |
| `bitset<N>` | DP bitmasks, sieve |

---

## Essential algorithms

```cpp
sort(v.begin(), v.end());
sort(v.begin(), v.end(), greater<>());
reverse(v.begin(), v.end());
auto it = lower_bound(v.begin(), v.end(), x);   // first >= x
auto jt = upper_bound(v.begin(), v.end(), x);   // first > x
binary_search(v.begin(), v.end(), x);
auto mxi = max_element(v.begin(), v.end());
auto mni = min_element(v.begin(), v.end());
long long s = accumulate(v.begin(), v.end(), 0LL);
next_permutation(v.begin(), v.end());
v.erase(unique(v.begin(), v.end()), v.end());
rotate(v.begin(), v.begin() + k, v.end());
nth_element(v.begin(), v.begin() + k, v.end());
partial_sum(v.begin(), v.end(), v.begin());
iota(v.begin(), v.end(), 0);
```

---

## I/O speedup

```cpp
int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    // ...
}
```
Drop both lines if you mix `cin` with `scanf`. Don't mix.

---

## `bitset<N>`

```cpp
bitset<64> b("1010");
b.set(5);            // bit 5 -> 1
b.reset(2);          // bit 2 -> 0
b.flip(0);
b[10];               // proxy ref
b.count();           // number of 1s — popcount
b.size();            // 64
b.any(); b.none(); b.all();
auto str = b.to_string();
auto u = b.to_ullong();

// Bitwise ops between two bitsets of same size
bitset<8> x("1100"), y("1010");
auto z = x | y;      // "1110"
auto w = x & y;      // "1000"
```

**Use case:** subset-sum DP, prime sieve, fast set ops over a known universe.

---

## Standard patterns

### Frequency / counter
```cpp
unordered_map<int,int> cnt;
for (int x : v) cnt[x]++;
```

### Top K frequent
```cpp
// O(n log k) using min-heap of size k
priority_queue<pair<int,int>, vector<pair<int,int>>, greater<>> pq;
for (auto& [val, c] : cnt) {
    pq.push({c, val});
    if ((int)pq.size() > K) pq.pop();
}
vector<int> top;
while (!pq.empty()) { top.push_back(pq.top().second); pq.pop(); }
reverse(top.begin(), top.end());
```

### Merge intervals
```cpp
sort(intervals.begin(), intervals.end());
vector<pair<int,int>> merged;
for (auto& [l, r] : intervals) {
    if (!merged.empty() && l <= merged.back().second)
        merged.back().second = max(merged.back().second, r);
    else
        merged.push_back({l, r});
}
```

### Kth largest with `nth_element`
```cpp
nth_element(v.begin(), v.begin() + (k-1), v.end(), greater<>());
int kth = v[k-1];
```

### Sliding window max — see [22 Idioms](22_idioms.md).

### BFS
```cpp
queue<int> q;
vector<int> dist(n, -1);
q.push(src); dist[src] = 0;
while (!q.empty()) {
    int u = q.front(); q.pop();
    for (int v : g[u]) if (dist[v] == -1) {
        dist[v] = dist[u] + 1;
        q.push(v);
    }
}
```

### DFS (iterative with stack)
```cpp
stack<int> st;
vector<int> seen(n);
st.push(src);
while (!st.empty()) {
    int u = st.top(); st.pop();
    if (seen[u]) continue;
    seen[u] = 1;
    for (int v : g[u]) if (!seen[v]) st.push(v);
}
```

### Dijkstra — see [05 Container Adapters](05_container_adapters.md).

### Two-sum (sorted)
```cpp
sort(a.begin(), a.end());
int l = 0, r = a.size() - 1;
while (l < r) {
    int s = a[l] + a[r];
    if (s == T) return {l, r};
    s < T ? ++l : --r;
}
```

### Longest substring without repeating chars
```cpp
unordered_map<char,int> last;
int best = 0, l = 0;
for (int r = 0; r < (int)s.size(); ++r) {
    auto it = last.find(s[r]);
    if (it != last.end() && it->second >= l) l = it->second + 1;
    last[s[r]] = r;
    best = max(best, r - l + 1);
}
```

### Median from a data stream (two heaps)
```cpp
priority_queue<int> lo;                                    // max-heap
priority_queue<int, vector<int>, greater<>> hi;            // min-heap

auto add = [&](int x){
    if (lo.empty() || x <= lo.top()) lo.push(x);
    else hi.push(x);
    if (lo.size() > hi.size() + 1) { hi.push(lo.top()); lo.pop(); }
    if (hi.size() > lo.size())     { lo.push(hi.top()); hi.pop(); }
};
auto median = [&]() -> double {
    if (lo.size() == hi.size()) return (lo.top() + hi.top()) / 2.0;
    return lo.top();
};
```

### Next permutation enumeration
```cpp
sort(v.begin(), v.end());
do {
    // process v
} while (next_permutation(v.begin(), v.end()));
```

---

## Anti-patterns to avoid in CP

- `accumulate(v.begin(), v.end(), 0)` when sum > 2^31. Use `0LL`.
- `unordered_map<int,int>` on adversarial input — use custom hash (see [21](21_hashing.md)).
- `pow()` for integer powers — write your own loop or `lround(pow(...))` only if you must.
- `endl` in tight loops — use `"\n"`.
- Recomputing `v.size()` inside hot loops if you don't need to (compiler usually handles it, but `int n = v.size()` is clearer).

## See also
- [22 Idioms](22_idioms.md)
- [24 Complexity](24_complexity.md)
