# 03 — Associative Containers

> Ordered, tree-based: `set`, `multiset`, `map`, `multimap`. All operations O(log n).

## Setup
```cpp
#include <bits/stdc++.h>
using namespace std;
```

---

## `std::set` — unique, sorted

```cpp
set<int> s = {3, 1, 4, 1, 5};       // stored as {1, 3, 4, 5}
s.insert(2);
s.erase(4);
auto it = s.find(3);
bool present = s.count(3);          // 0 or 1

for (int x : s) cout << x << ' ';   // => 1 2 3 5
```

### `lower_bound` / `upper_bound` / `equal_range`
```cpp
set<int> s = {10, 20, 30, 40};
auto lb = s.lower_bound(20);   // points to 20 (first >= 20)
auto ub = s.upper_bound(20);   // points to 30 (first > 20)
auto [b, e] = s.equal_range(20);   // C++17 structured binding
```
**Important:** use member `s.lower_bound(x)`, not `std::lower_bound(s.begin(), s.end(), x)` — the latter is O(n) on bidirectional iterators.

### Custom comparator
```cpp
set<int, greater<int>> desc = {1, 2, 3};   // stored as {3, 2, 1}

struct ByAbs {
    bool operator()(int a, int b) const { return abs(a) < abs(b); }
};
set<int, ByAbs> s2 = {-3, 1, -2};   // ordered by |x|
```

**Iterator stability:** insert never invalidates; erase only invalidates the erased iterator.

---

## `std::multiset` — duplicates allowed, sorted

```cpp
multiset<int> ms = {1, 2, 2, 2, 3};
ms.count(2);                       // 3
ms.erase(2);                       // removes ALL 2s
ms.erase(ms.find(2));              // removes one occurrence

auto [b, e] = ms.equal_range(2);
for (auto it = b; it != e; ++it) cout << *it << ' ';
```

**Use when:** you need sorted duplicates and range queries by key.

---

## `std::map` — key→value, unique sorted keys

```cpp
map<string, int> freq;
freq["apple"] = 1;
freq["banana"] = 2;
freq["apple"]++;             // freq["apple"] becomes 2

for (auto& [k, v] : freq)    // C++17 structured binding
    cout << k << "->" << v << ' ';
```

### `operator[]` vs `at` vs `find`
```cpp
map<string,int> m;
m["x"];               // INSERTS "x" -> 0 if absent
m.at("y");            // THROWS std::out_of_range if absent
auto it = m.find("y");
if (it != m.end()) cout << it->second;
```
**Gotcha:** `m[key]` inserts a default value if missing. To peek without inserting, use `find` or `count`.

### Insertion variants
```cpp
m.insert({"k", 1});
m.insert(make_pair("k", 1));
m.emplace("k", 1);           // construct in place
auto [it, inserted] = m.insert({"k", 99});   // inserted=false if "k" existed
```

### Range queries
```cpp
map<int, string> tbl = {{10,"a"}, {20,"b"}, {30,"c"}};
auto lb = tbl.lower_bound(15);   // points to {20,"b"}
auto ub = tbl.upper_bound(20);   // points to {30,"c"}
```

### Custom key ordering
```cpp
map<string, int, greater<string>> m;   // reverse lexicographic
```

**Iterator stability:** same as `set`.

---

## `std::multimap` — key→value, duplicate keys allowed

```cpp
multimap<string, int> mm;
mm.insert({"a", 1});
mm.insert({"a", 2});
mm.insert({"a", 3});
mm.insert({"b", 9});

auto [b, e] = mm.equal_range("a");
for (auto it = b; it != e; ++it)
    cout << it->first << "->" << it->second << ' ';
// => a->1 a->2 a->3
```

**No `operator[]`** — ambiguous since one key may map to many values.

**Use when:** one key maps to multiple values and you want them ordered.

---

## When to use which

| Need | Use |
|---|---|
| Unique sorted values | `set` |
| Sorted duplicates | `multiset` |
| Sorted key→value | `map` |
| Sorted key→many values | `multimap` |
| O(1) lookups, no order needed | [unordered_*](04_unordered_containers.md) |

## See also
- [04 Unordered Containers](04_unordered_containers.md)
- [20 Comparators](20_comparators.md) for custom orderings
- [24 Complexity](24_complexity.md)
