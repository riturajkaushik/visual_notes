# 02 — Sequence Containers

> `vector`, `array`, `deque`, `list`, `forward_list`.

## Setup
```cpp
#include <bits/stdc++.h>
using namespace std;
```

---

## `std::vector` — dynamic contiguous array

### Construction
```cpp
vector<int> a;                       // empty
vector<int> b(5);                    // {0,0,0,0,0}
vector<int> c(5, 7);                 // {7,7,7,7,7}
vector<int> d = {1, 2, 3};           // initializer list
vector<int> e(d.begin(), d.end());   // from range
vector<vector<int>> grid(3, vector<int>(4, 0));   // 3x4 matrix of 0s
```

### Add / remove
```cpp
vector<int> v;
v.push_back(10);        // copy/move into back
v.emplace_back(20);     // construct in place
v.pop_back();           // remove last
v.insert(v.begin() + 1, 99);
v.erase(v.begin());                       // remove first
v.erase(v.begin(), v.begin() + 2);        // remove range
v.clear();
```
**Complexity:** `push_back` amortized O(1); `insert`/`erase` in middle O(n).

### Access
```cpp
v[0];           // no bounds check
v.at(0);        // throws std::out_of_range
v.front(); v.back();
int* p = v.data();      // raw pointer to contiguous storage
```

### Size vs capacity
```cpp
vector<int> v;
v.reserve(1000);   // allocate capacity, size stays 0 — avoids reallocation
v.resize(5);       // size becomes 5; new elements default-constructed
v.resize(10, -1);  // grow to 10, fill new slots with -1
v.shrink_to_fit(); // hint: trim capacity
cout << v.size() << " " << v.capacity();
```

### Iteration
```cpp
for (int x : v) cout << x << ' ';
for (auto it = v.begin(); it != v.end(); ++it) cout << *it;
sort(v.begin(), v.end());
```

### Iterator invalidation (read this)
- `push_back`, `emplace_back`, `insert`, `reserve`, `resize`: **all iterators invalidated if reallocation happens**.
- `erase`: iterators at and after the erased position invalidated.
- No reallocation if new size ≤ capacity.

**Gotcha:** never hold an iterator across a `push_back` unless you `reserve()`d first.

### When to use
Default sequence container. Use unless you have a specific reason not to.

---

## `std::array` — fixed-size, stack-allocated

```cpp
array<int, 5> a = {1, 2, 3, 4, 5};
a.size();           // 5 (constexpr)
a.front(); a.back();
int* p = a.data();
sort(a.begin(), a.end());

// Difference from C-array: knows its size, decays nicely, has STL iterators.
array<int, 3> x{10, 20, 30};
for (int v : x) cout << v << ' ';   // => 10 20 30
```
**Use when:** size known at compile time and you want STL compatibility.

---

## `std::deque` — double-ended queue

```cpp
deque<int> d;
d.push_back(3);
d.push_front(1);
d.push_back(4);     // d = {1, 3, 4}
d.pop_front();      // d = {3, 4}
d[0];               // random access OK
```
**Complexity:** `push_front/back`, `pop_front/back` O(1); random access O(1) but slower than `vector`.

**Memory model:** segmented (array of fixed-size blocks). Not contiguous — `data()` does not exist.

**Iterator invalidation:** any insert/erase at the ends invalidates iterators (pointers/references to elements may stay valid for end operations, but iterators don't).

**Use when:** you need fast insertion/removal at both ends.

---

## `std::list` — doubly linked list

```cpp
list<int> L = {1, 2, 3};
L.push_back(4);
L.push_front(0);
auto it = L.begin();
advance(it, 2);
L.insert(it, 99);       // O(1) with iterator
L.erase(it);            // O(1)

L.sort();               // O(n log n), uses its own merge sort
L.reverse();
L.unique();             // remove consecutive duplicates
```

### `splice` — move nodes between lists in O(1)
```cpp
list<int> a = {1, 2, 3};
list<int> b = {10, 20};
a.splice(a.end(), b);   // a = {1,2,3,10,20}, b is empty
```

### `merge`
```cpp
list<int> x = {1, 3, 5};
list<int> y = {2, 4, 6};
x.merge(y);             // x = {1,2,3,4,5,6}, y empty (both must be sorted)
```

**Iterator stability:** insert/erase never invalidates other iterators.
**No random access.** No `operator[]`, no `O(1)` index lookup.

**Use when:** frequent middle insertion/deletion AND iterators must stay valid. Rarely the right choice — usually `vector` wins due to cache locality.

---

## `std::forward_list` — singly linked list

```cpp
forward_list<int> fl = {1, 2, 3};
fl.push_front(0);                 // O(1)
auto it = fl.before_begin();      // "iterator before first"
fl.insert_after(it, 99);          // insert at front => {99,0,1,2,3}
fl.erase_after(it);               // erases the 99
fl.remove(2);                     // remove all 2s
```
- No `size()` member (C++11/14). C++17 still doesn't add one — by design, keeps overhead minimal.
- Lowest memory overhead of any STL container.

**Use when:** memory is extremely constrained and singly linked is acceptable.

---

## Quick chooser

| Need | Use |
|---|---|
| Default | `vector` |
| Fixed compile-time size | `array` |
| Insert/remove at both ends | `deque` |
| Frequent middle insert AND stable iterators | `list` |
| Smallest linked-list overhead | `forward_list` |

## See also
- [03 Associative](03_associative_containers.md) · [05 Adapters](05_container_adapters.md)
- [07 Iterators](07_iterators.md) — invalidation rules in depth
- [24 Complexity](24_complexity.md)
