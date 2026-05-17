# 04 — Unordered Containers

> Hash-table-based: `unordered_set`, `unordered_multiset`, `unordered_map`, `unordered_multimap`. Average O(1) ops.

## Setup
```cpp
#include <bits/stdc++.h>
using namespace std;
```

---

## `std::unordered_set`

```cpp
unordered_set<int> us = {1, 2, 3, 1};   // {1, 2, 3} (order arbitrary)
us.insert(4);
us.erase(2);
us.count(3);             // 0 or 1
auto it = us.find(3);
for (int x : us) cout << x << ' ';  // arbitrary order
```
**Complexity:** insert/erase/find average O(1), worst O(n) on collisions.

---

## `std::unordered_multiset`

```cpp
unordered_multiset<int> ums = {1, 1, 2, 3, 3, 3};
ums.count(3);                        // 3
ums.erase(3);                        // removes ALL 3s
ums.erase(ums.find(3));              // removes one

auto [b, e] = ums.equal_range(3);
for (auto it = b; it != e; ++it) cout << *it << ' ';
```

---

## `std::unordered_map`

```cpp
unordered_map<string, int> mp;
mp["cat"] = 1;
mp["dog"]++;             // inserts dog=0 then increments to 1
mp.at("cat");            // throws if missing
auto it = mp.find("dog");
if (it != mp.end()) cout << it->second;

for (auto& [k, v] : mp)  // C++17
    cout << k << ':' << v << ' ';
```

### Frequency counting idiom
```cpp
vector<string> words = {"a", "b", "a", "c", "b", "a"};
unordered_map<string, int> freq;
for (const auto& w : words) freq[w]++;
// freq = {"a":3, "b":2, "c":1}
```

---

## `std::unordered_multimap`

```cpp
unordered_multimap<string, int> umm;
umm.insert({"k", 1});
umm.insert({"k", 2});
auto [b, e] = umm.equal_range("k");
for (auto it = b; it != e; ++it) cout << it->second << ' ';
```

---

## Hashing internals you should know

```cpp
unordered_map<int, int> m;
m.bucket_count();        // number of buckets
m.load_factor();         // size / bucket_count
m.max_load_factor();     // default 1.0; growth triggered above this
m.rehash(1024);          // ensure at least 1024 buckets
m.reserve(10000);        // ensure no rehash needed for 10000 insertions
```

**Iterator invalidation:** any insertion that causes a rehash invalidates **all** iterators (but pointers/references to elements stay valid). Erase only invalidates the erased iterator.

**Performance tip:** `reserve(n)` before bulk inserts to avoid repeated rehashing.

---

## Custom hash & equality

Required for keys that don't have a default `std::hash` (e.g. `pair<int,int>`, your own structs).

### For `pair<int,int>`
```cpp
struct PairHash {
    size_t operator()(const pair<int,int>& p) const noexcept {
        return hash<long long>()(((long long)p.first << 32) ^ (unsigned)p.second);
    }
};
unordered_map<pair<int,int>, int, PairHash> grid;
grid[{0, 0}] = 1;
```

### For a struct
```cpp
struct Point { int x, y; };
struct PointHash {
    size_t operator()(const Point& p) const noexcept {
        return hash<int>()(p.x) * 31 ^ hash<int>()(p.y);
    }
};
struct PointEq {
    bool operator()(const Point& a, const Point& b) const noexcept {
        return a.x == b.x && a.y == b.y;
    }
};
unordered_set<Point, PointHash, PointEq> seen;
```

More on hashing strategies: [21 Hashing](21_hashing.md).

---

## ordered vs unordered

| | `set` / `map` | `unordered_set` / `unordered_map` |
|---|---|---|
| Order | sorted | unordered |
| insert/find/erase | O(log n) | avg O(1), worst O(n) |
| Range queries (`lower_bound`) | yes | no |
| Memory overhead | lower | higher (buckets) |
| Default for keys | works for any `<`-comparable | needs `std::hash` |
| Adversarial input | safe | can hit worst case (CP: use custom hash) |

## See also
- [21 Hashing](21_hashing.md) for collision-safe custom hash for CP
- [24 Complexity](24_complexity.md)
