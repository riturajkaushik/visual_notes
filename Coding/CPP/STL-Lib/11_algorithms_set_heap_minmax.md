# 11 — Set Operations, Heap Algorithms, Min/Max

> Three small but useful algorithm families from `<algorithm>`.

## Setup
```cpp
#include <bits/stdc++.h>
using namespace std;
```

---

# Set Operations (on sorted ranges)

Inputs must be sorted. Output is sorted.

## `includes` — subset test
```cpp
vector<int> a = {1, 2, 3, 4, 5};
vector<int> b = {2, 4};
includes(a.begin(), a.end(), b.begin(), b.end());   // true
```

## `set_union`
```cpp
vector<int> a = {1, 2, 3}, b = {2, 3, 4}, out;
set_union(a.begin(), a.end(), b.begin(), b.end(), back_inserter(out));
// out = {1, 2, 3, 4}
```

## `set_intersection`
```cpp
vector<int> a = {1, 2, 3, 4}, b = {2, 4, 6}, out;
set_intersection(a.begin(), a.end(), b.begin(), b.end(), back_inserter(out));
// out = {2, 4}
```

## `set_difference` — a \ b
```cpp
vector<int> a = {1, 2, 3, 4}, b = {2, 4}, out;
set_difference(a.begin(), a.end(), b.begin(), b.end(), back_inserter(out));
// out = {1, 3}
```

## `set_symmetric_difference` — XOR
```cpp
vector<int> a = {1, 2, 3}, b = {2, 3, 4}, out;
set_symmetric_difference(a.begin(), a.end(), b.begin(), b.end(), back_inserter(out));
// out = {1, 4}
```

**Complexity:** O(n + m).
**Trap:** unsorted input → wrong output, no error.

---

# Heap Algorithms (raw heap on a range)

A heap is maintained on any random-access range. By default: max-heap.

## `make_heap`
```cpp
vector<int> v = {3, 1, 4, 1, 5, 9, 2, 6};
make_heap(v.begin(), v.end());
// v[0] is the largest (9)
```
**Complexity:** O(n).

## `push_heap` — after appending, sift up
```cpp
v.push_back(100);
push_heap(v.begin(), v.end());   // O(log n)
// v[0] now 100
```

## `pop_heap` — move max to end, restore heap on the prefix
```cpp
pop_heap(v.begin(), v.end());    // O(log n)
int top = v.back();
v.pop_back();
```

## `sort_heap` — turn heap into sorted range
```cpp
make_heap(v.begin(), v.end());
sort_heap(v.begin(), v.end());   // ascending, since it's a max-heap
```
**Complexity:** O(n log n).

## `is_heap`, `is_heap_until`
```cpp
is_heap(v.begin(), v.end());
auto it = is_heap_until(v.begin(), v.end());
```

## Min-heap via comparator
```cpp
make_heap(v.begin(), v.end(), greater<int>());
push_heap(v.begin(), v.end(), greater<int>());
pop_heap (v.begin(), v.end(), greater<int>());
```
**Gotcha:** the *same* comparator must be passed to every heap call on the range.

## When to use raw heap vs `priority_queue`
- `priority_queue` is the friendly wrapper — use by default.
- Use raw heap when you need to iterate the underlying storage, swap heaps cheaply, or convert heap→sorted in place (`sort_heap`).

---

# Min / Max

## `min`, `max`
```cpp
min(3, 7);                  // 3
max(3, 7);                  // 7
min({5, 1, 3, 2});          // 1   (initializer list overload)
max({5, 1, 3, 2});          // 5
```

## `minmax` — both at once
```cpp
auto [lo, hi] = minmax({4, 1, 9, 2});   // C++17 structured binding
// lo = 1, hi = 9
```

## `min_element`, `max_element`, `minmax_element` — work on ranges
```cpp
vector<int> v = {4, 1, 9, 2, 7};
auto it_min = min_element(v.begin(), v.end());   // -> 1
auto it_max = max_element(v.begin(), v.end());   // -> 9
auto [lo_it, hi_it] = minmax_element(v.begin(), v.end());

// custom comparator: longest string
vector<string> ws = {"a", "bb", "ccc", "d"};
auto longest = max_element(ws.begin(), ws.end(),
                           [](const string& a, const string& b){
                               return a.size() < b.size();
                           });
// longest -> "ccc"
```

## `clamp` (C++17)
```cpp
clamp(5, 0, 10);     // 5
clamp(-3, 0, 10);    // 0
clamp(99, 0, 10);    // 10
// with custom comparator
clamp(7, 10, 5, greater<int>());   // odd: when lo>hi UB, here lo=10 > hi=5 inverted
```
**Complexity:** O(1).
**Use for:** snapping into a range without writing `max(lo, min(hi, x))`.

## See also
- [05 Container Adapters](05_container_adapters.md) — `priority_queue`
- [10 Sorting & Search](10_algorithms_sorting_searching.md)
- [20 Comparators](20_comparators.md)
