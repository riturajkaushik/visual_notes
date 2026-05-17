# 10 — Sorting & Binary Search

> `sort`, `stable_sort`, `nth_element`, `partial_sort`, plus the `_bound` family for sorted ranges.

## Setup
```cpp
#include <bits/stdc++.h>
using namespace std;
```

---

## `sort`

```cpp
vector<int> v = {3, 1, 4, 1, 5, 9, 2, 6};
sort(v.begin(), v.end());                // {1,1,2,3,4,5,6,9}
sort(v.begin(), v.end(), greater<int>()); // {9,6,5,4,3,2,1,1}
```
**Complexity:** O(n log n). Not stable — equal keys can be reordered.

### Custom comparator (lambda)
```cpp
vector<pair<string,int>> v = {{"a",2},{"b",1},{"c",2}};
sort(v.begin(), v.end(),
     [](const auto& a, const auto& b){ return a.second < b.second; });
```
**Strict weak ordering:** `cmp(a,b) && cmp(b,a)` must never both be true. Use `<` semantics. Returning `<=` will crash or loop on some libcxx builds.

### Multi-key sort with `tie`
```cpp
struct Row { int dept; int score; string name; };
vector<Row> rows = /*...*/;
sort(rows.begin(), rows.end(), [](const Row& a, const Row& b){
    return tie(a.dept, b.score, a.name)    // careful with ordering desired
         < tie(b.dept, a.score, b.name);
});
// Simpler when all ascending:
sort(rows.begin(), rows.end(), [](const Row& a, const Row& b){
    return tie(a.dept, a.score, a.name) < tie(b.dept, b.score, b.name);
});
```

---

## `stable_sort` — preserves order of equal keys

```cpp
vector<pair<string,int>> v = {{"a",1},{"b",2},{"a",2},{"b",1}};
stable_sort(v.begin(), v.end(),
            [](const auto& x, const auto& y){ return x.first < y.first; });
// a,1  a,2  b,2  b,1   — equal-key relative order preserved
```
**Complexity:** O(n log n) (may use extra memory).

---

## `partial_sort` — top-K sorted at the front

```cpp
vector<int> v = {5, 3, 8, 1, 9, 2, 7};
partial_sort(v.begin(), v.begin() + 3, v.end());
// First 3 elements are the 3 smallest, sorted: {1,2,3, ?, ?, ?, ?}
```
**Complexity:** O(n log k).

### `partial_sort_copy`
```cpp
vector<int> src = {5, 3, 8, 1, 9, 2, 7};
vector<int> dst(3);
partial_sort_copy(src.begin(), src.end(), dst.begin(), dst.end());
// dst = {1, 2, 3}, src unchanged
```

---

## `nth_element` — quickselect

Puts the n-th element in its sorted position; everything before is `<=` it, everything after is `>=` it. Order otherwise unspecified.

```cpp
vector<int> v = {5, 3, 8, 1, 9, 2, 7};
nth_element(v.begin(), v.begin() + 3, v.end());
// v[3] is the 4th smallest = 5
// v could be {1,2,3, 5, 7,8,9} or {3,1,2, 5, 9,7,8} — only v[3] guaranteed
```
**Complexity:** average O(n). Use for k-th smallest / largest without full sort.

### k-th largest
```cpp
auto k = 3;
nth_element(v.begin(), v.begin() + (k-1), v.end(), greater<int>());
int kth_largest = v[k-1];
```

---

## `is_sorted` / `is_sorted_until`

```cpp
vector<int> v = {1, 2, 3, 5, 4, 6};
is_sorted(v.begin(), v.end());                  // false
auto it = is_sorted_until(v.begin(), v.end());  // points to 4
```

---

# Binary Search Family

**All require a sorted range.**

## `binary_search` — yes/no

```cpp
vector<int> v = {10, 20, 30, 40};
binary_search(v.begin(), v.end(), 20);   // true
binary_search(v.begin(), v.end(), 25);   // false
```
**Complexity:** O(log n) on random access; O(n) on bidirectional (still does log compares but linear advances).

---

## `lower_bound` — first `>= x`

```cpp
vector<int> v = {1, 3, 3, 3, 5, 7};
auto it = lower_bound(v.begin(), v.end(), 3);
// it points to the first 3 (index 1)
int idx = it - v.begin();      // 1
```

## `upper_bound` — first `> x`

```cpp
auto it = upper_bound(v.begin(), v.end(), 3);
// it points to 5 (index 4)
```

## `equal_range` — `[lower_bound, upper_bound)`

```cpp
auto [b, e] = equal_range(v.begin(), v.end(), 3);
int count_of_3 = e - b;   // 3
```

---

## Useful patterns

### Index of first element `>= x`
```cpp
int idx = lower_bound(v.begin(), v.end(), x) - v.begin();
```

### Closest element to x (in sorted v)
```cpp
auto it = lower_bound(v.begin(), v.end(), x);
if (it == v.end()) /* answer is v.back() */;
else if (it == v.begin()) /* answer is v.front() */;
else {
    int hi = *it, lo = *prev(it);
    int answer = (x - lo <= hi - x) ? lo : hi;
}
```

### Count occurrences in sorted vector
```cpp
auto [b, e] = equal_range(v.begin(), v.end(), x);
size_t cnt = e - b;
```

### Custom comparator binary search
The `cmp` you pass must be the SAME ordering the range is sorted with.
```cpp
sort(v.begin(), v.end(), greater<int>());
auto it = lower_bound(v.begin(), v.end(), 5, greater<int>());
```

---

## `set` / `map` member `lower_bound`

```cpp
set<int> s = {10, 20, 30};
auto it = s.lower_bound(15);   // points to 20, O(log n)

// DON'T do this — generic algorithm is O(n) on bidirectional iterators:
auto bad = std::lower_bound(s.begin(), s.end(), 15);
```

## See also
- [11 Set, Heap, Min/Max](11_algorithms_set_heap_minmax.md)
- [20 Comparators](20_comparators.md)
- [22 Idioms](22_idioms.md) — coordinate compression
