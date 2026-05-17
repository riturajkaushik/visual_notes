# 07 — Iterators

> Generalized pointers. Algorithms operate on ranges `[first, last)`.

## Setup
```cpp
#include <bits/stdc++.h>
using namespace std;
```

---

## Basics

```cpp
vector<int> v = {10, 20, 30, 40};

auto it = v.begin();    // points to 10
*it;                    // 10
++it;                   // points to 20
it - v.begin();         // 1 (only on random-access iterators)
auto e = v.end();       // ONE PAST the last element — do not dereference
```

### Half-open range `[first, last)`
- `first` is included, `last` is **not**.
- Length = `last - first`.
- An empty range is `first == last`.

### Iteration variants
```cpp
for (auto it = v.begin(); it != v.end(); ++it) cout << *it << ' ';
for (auto it = v.cbegin(); it != v.cend(); ++it) cout << *it << ' ';   // const
for (auto it = v.rbegin(); it != v.rend(); ++it) cout << *it << ' ';   // reverse
for (int x : v) cout << x << ' ';            // range-based for
```

### const iterators
```cpp
vector<int> v = {1,2,3};
auto       it1 = v.begin();    // iterator to int
auto       it2 = v.cbegin();   // const_iterator — *it2 is const int&
const auto it3 = v.begin();    // pointer is const; *it3 is still int&
```

### Reverse iterators
```cpp
for (auto it = v.rbegin(); it != v.rend(); ++it) cout << *it;
// rbegin() points to last element; rend() is "before first"
// Convert: rit.base() points one past the element rit points to
```

---

## Iterator Categories

From weakest to strongest:

| Category | Can | Containers |
|---|---|---|
| Input | read once, ++ | `istream_iterator` |
| Output | write once, ++ | `ostream_iterator`, `back_inserter` |
| Forward | read/write, ++ | `forward_list`, `unordered_*` |
| Bidirectional | also `--` | `list`, `set`, `map`, `multiset`, `multimap` |
| Random access | also `+n`, `-n`, `it[n]`, `<`, `>` | `vector`, `deque`, `array`, raw arrays |
| Contiguous (C++17 formalized) | random access + storage is contiguous | `vector`, `array`, raw arrays |

**Why it matters:** algorithms have minimum iterator requirements.
- `std::sort` needs random access — won't compile on `list`. Use `list::sort()` instead.
- `std::lower_bound` *compiles* on any forward iterator but is O(n) without random access.

---

## Iterator Utilities

```cpp
list<int> L = {10, 20, 30, 40, 50};
auto it = L.begin();

advance(it, 3);         // it now points to 40
auto j = next(it);      // points to 50; doesn't modify it
auto k = prev(it);      // points to 30
distance(L.begin(), it); // 3
```

### Inserters — let algorithms grow a container
```cpp
vector<int> src = {1, 2, 3};
vector<int> dst;
copy(src.begin(), src.end(), back_inserter(dst));   // dst = {1,2,3}

list<int> lst;
copy(src.begin(), src.end(), front_inserter(lst));  // lst = {3,2,1} (each inserted at front)

set<int> s;
copy(src.begin(), src.end(), inserter(s, s.end()));
```

### Stream iterators
```cpp
// Read all ints from cin until EOF
vector<int> v((istream_iterator<int>(cin)),
              istream_iterator<int>());

// Print with separator
copy(v.begin(), v.end(), ostream_iterator<int>(cout, " "));
```

### `make_move_iterator` — move elements into destination
```cpp
vector<string> src = {"a", "b", "c"};
vector<string> dst;
copy(make_move_iterator(src.begin()),
     make_move_iterator(src.end()),
     back_inserter(dst));
// src elements are now moved-from (valid but unspecified)
```

---

## Iterator Invalidation — must-know table

| Container | Insert invalidates | Erase invalidates |
|---|---|---|
| `vector` | all iters if reallocation; else iters at/after position | iters at/after erased position |
| `deque` | all iters | all iters (except `begin()`/`end()` for end ops, but iters in middle die) |
| `list` / `forward_list` | none | only the erased iter |
| `set`/`map`/`multi*` | none | only the erased iter |
| `unordered_*` | all iters if rehash (load_factor threshold) | only the erased iter; pointers/refs to elements survive |

**Common bug:**
```cpp
for (auto it = v.begin(); it != v.end(); ++it)
    if (*it == 0) v.erase(it);    // UB: it invalidated, then ++it
```
**Fix — use erase's return value:**
```cpp
for (auto it = v.begin(); it != v.end(); )
    if (*it == 0) it = v.erase(it);
    else          ++it;
```
**Or use erase-remove:**
```cpp
v.erase(remove(v.begin(), v.end(), 0), v.end());
```

---

## Iterator traits (peek)
```cpp
using it_t = vector<int>::iterator;
using cat  = iterator_traits<it_t>::iterator_category;
using val  = iterator_traits<it_t>::value_type;     // int
```
More: [26 Advanced Topics](26_advanced_topics.md).

## See also
- [02 Sequence Containers](02_sequence_containers.md) — container-specific invalidation
- [04 Unordered Containers](04_unordered_containers.md) — rehash invalidation
- [22 Idioms](22_idioms.md)
