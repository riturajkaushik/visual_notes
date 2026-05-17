# 12 — Permutations & Partitioning

> Lexicographic permutations and partitioning a range by a predicate. From `<algorithm>`.

## Setup
```cpp
#include <bits/stdc++.h>
using namespace std;
```

---

# Permutations

## `next_permutation` — next lex order

```cpp
vector<int> v = {1, 2, 3};
do {
    for (int x : v) cout << x;
    cout << ' ';
} while (next_permutation(v.begin(), v.end()));
// => 123 132 213 231 312 321
```

**Returns:** `true` if a next permutation existed; `false` if the range was at the highest permutation (in which case it wraps to the lowest).

**Important:** start from the **sorted** range to enumerate ALL permutations.

```cpp
sort(v.begin(), v.end());
do {
    // ...
} while (next_permutation(v.begin(), v.end()));
```

## `prev_permutation` — previous lex order

```cpp
vector<int> v = {3, 2, 1};
do {
    for (int x : v) cout << x;
    cout << ' ';
} while (prev_permutation(v.begin(), v.end()));
// => 321 312 231 213 132 123
```

## `is_permutation`
```cpp
vector<int> a = {1, 2, 3, 4};
vector<int> b = {4, 1, 3, 2};
is_permutation(a.begin(), a.end(), b.begin());   // true
```
**Complexity:** O(n²) worst case (better with hash for unique elements — write your own).

## With duplicates
`next_permutation` skips duplicates correctly when starting sorted:
```cpp
vector<int> v = {1, 1, 2};
do { /* prints 112, 121, 211 only */ } while (next_permutation(v.begin(), v.end()));
```

---

# Partitioning

A partition splits a range into two groups by predicate: matches first, non-matches after (or vice versa for some algos).

## `partition` — fastest, no order guarantee
```cpp
vector<int> v = {1, 2, 3, 4, 5, 6};
auto mid = partition(v.begin(), v.end(), [](int x){ return x % 2 == 0; });
// Evens come first, then odds. Order within each group is unspecified.
// e.g. v might be {6, 2, 4, 3, 5, 1} or similar.
// mid points to the first odd element.
```
**Complexity:** O(n).

## `stable_partition` — preserves relative order within groups
```cpp
vector<int> v = {1, 2, 3, 4, 5, 6};
auto mid = stable_partition(v.begin(), v.end(), [](int x){ return x % 2 == 0; });
// v = {2, 4, 6, 1, 3, 5}
// mid points to 1
```
**Complexity:** O(n log n) without extra memory, O(n) with.

## `is_partitioned`
```cpp
vector<int> v = {2, 4, 6, 1, 3, 5};
is_partitioned(v.begin(), v.end(), [](int x){ return x % 2 == 0; });   // true
```

## `partition_point` — binary search for the partition boundary
Requires a *partitioned* range with respect to the predicate.
```cpp
vector<int> v = {2, 4, 6, 1, 3, 5};
auto mid = partition_point(v.begin(), v.end(), [](int x){ return x % 2 == 0; });
// mid points to 1
```
**Complexity:** O(log n).

This is the generic form of `lower_bound`: `lower_bound(v, x)` is essentially `partition_point(v, [&](int e){ return e < x; })`.

## `partition_copy` — write two outputs
```cpp
vector<int> v = {1, 2, 3, 4, 5, 6};
vector<int> evens, odds;
partition_copy(v.begin(), v.end(),
               back_inserter(evens), back_inserter(odds),
               [](int x){ return x % 2 == 0; });
// evens = {2,4,6}, odds = {1,3,5}, v unchanged
```

---

## Use cases

| Pattern | Use |
|---|---|
| Generate all permutations | `next_permutation` after `sort` |
| Check anagram | `is_permutation` (or sort and compare) |
| Quickselect-like split | `partition` |
| Group-by while preserving order | `stable_partition` |
| Binary search on monotone predicate | `partition_point` |
| Split into two vectors | `partition_copy` |

## See also
- [10 Sorting & Search](10_algorithms_sorting_searching.md)
- [22 Idioms](22_idioms.md)
