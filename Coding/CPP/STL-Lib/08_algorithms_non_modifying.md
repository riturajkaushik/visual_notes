# 08 — Non-Modifying Algorithms

> Algorithms that read from a range without changing element values. From `<algorithm>`.

## Setup
```cpp
#include <bits/stdc++.h>
using namespace std;
```

---

## Algorithm Design Principles
- Operate on iterator ranges `[first, last)`.
- Don't change container size; use erase-remove idiom for removal.
- Generic over iterator categories — same code, many containers.

---

## Quantifiers: `all_of`, `any_of`, `none_of`

```cpp
vector<int> v = {2, 4, 6, 8};
all_of (v.begin(), v.end(), [](int x){ return x % 2 == 0; });   // true
any_of (v.begin(), v.end(), [](int x){ return x > 5; });        // true
none_of(v.begin(), v.end(), [](int x){ return x < 0; });        // true
```
**Complexity:** O(n) each, short-circuits on first counterexample.

---

## `for_each` — apply a function to each element

```cpp
vector<int> v = {1, 2, 3};
for_each(v.begin(), v.end(), [](int x){ cout << x << ' '; });   // => 1 2 3

int sum = 0;
for_each(v.begin(), v.end(), [&](int x){ sum += x; });
// sum = 6
```
Range-for is usually cleaner; `for_each` is useful when you want to pass a named functor or with execution policies (C++17).

---

## Counting: `count`, `count_if`

```cpp
vector<int> v = {1, 2, 2, 3, 2};
count   (v.begin(), v.end(), 2);                          // 3
count_if(v.begin(), v.end(), [](int x){ return x > 1; }); // 4
```

---

## Searching: `find`, `find_if`, `find_if_not`

```cpp
vector<int> v = {10, 20, 30, 40};
auto it = find(v.begin(), v.end(), 30);
if (it != v.end()) cout << *it;        // 30

auto p = find_if(v.begin(), v.end(), [](int x){ return x > 25; });   // -> 30
auto q = find_if_not(v.begin(), v.end(), [](int x){ return x < 25; });   // -> 30
```
**Complexity:** O(n). For sorted ranges use `binary_search` / `lower_bound` (see [10](10_algorithms_sorting_searching.md)).

---

## Pair comparison: `mismatch`, `equal`

```cpp
vector<int> a = {1, 2, 3, 4};
vector<int> b = {1, 2, 9, 4};
auto [ia, ib] = mismatch(a.begin(), a.end(), b.begin());
// *ia == 3, *ib == 9

equal(a.begin(), a.end(), b.begin());                            // false
equal(a.begin(), a.begin() + 2, b.begin());                      // true
```

---

## Substring / subsequence search

### `search` — find a subsequence
```cpp
vector<int> hay = {1, 2, 3, 4, 5, 6};
vector<int> nee = {3, 4, 5};
auto it = search(hay.begin(), hay.end(), nee.begin(), nee.end());
// it points to the 3
```

### `find_end` — last occurrence of a subsequence
```cpp
vector<int> v = {1, 2, 3, 1, 2, 3};
vector<int> p = {1, 2};
auto it = find_end(v.begin(), v.end(), p.begin(), p.end());
// points to the second 1
```

### `find_first_of` — first element from a set of candidates
```cpp
vector<int> v = {10, 20, 30, 40};
vector<int> cand = {25, 30, 50};
auto it = find_first_of(v.begin(), v.end(), cand.begin(), cand.end());
// points to 30
```

### `adjacent_find` — first pair of equal consecutive elements
```cpp
vector<int> v = {1, 2, 2, 3, 3, 3};
auto it = adjacent_find(v.begin(), v.end());   // -> first 2
// custom predicate
auto it2 = adjacent_find(v.begin(), v.end(),
                         [](int a, int b){ return b == a + 1; });   // first 1,2
```

---

## Mini cheat sheet

| Goal | Use |
|---|---|
| Does every / any / none satisfy P? | `all_of` / `any_of` / `none_of` |
| Run f on each | `for_each` |
| Count matches | `count`, `count_if` |
| First match | `find`, `find_if`, `find_if_not` |
| First differing pair | `mismatch` |
| Are two ranges equal? | `equal` |
| Find a subsequence | `search` (first), `find_end` (last) |
| First element in candidate set | `find_first_of` |
| First adjacent duplicates | `adjacent_find` |

## See also
- [09 Modifying Algorithms](09_algorithms_modifying.md)
- [10 Sorting & Binary Search](10_algorithms_sorting_searching.md)
