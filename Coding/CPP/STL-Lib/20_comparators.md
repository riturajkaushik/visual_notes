# 20 — Comparators & Custom Ordering

> Custom orderings for `sort`, `set`/`map`, and `priority_queue`.

## Setup
```cpp
#include <bits/stdc++.h>
using namespace std;
```

---

## Strict weak ordering

A comparator `cmp(a, b)` must satisfy:
1. **Irreflexive:** `cmp(a, a) == false`.
2. **Asymmetric:** if `cmp(a, b)` then `!cmp(b, a)`.
3. **Transitive:** if `cmp(a, b)` and `cmp(b, c)` then `cmp(a, c)`.
4. **Transitivity of equivalence:** if `!cmp(a, b) && !cmp(b, a)` and same for `(b, c)`, then same for `(a, c)`.

**Practical rule:** return `<` (strictly less), never `<=`. `<=` violates irreflexivity and crashes some libcxx debug modes.

---

## Comparator for `sort`

### Lambda
```cpp
vector<int> v = {3, 1, 4, 1, 5};
sort(v.begin(), v.end(), [](int a, int b){ return a > b; });   // descending
```

### Sorting pairs (already lex by default; override if needed)
```cpp
vector<pair<int,string>> v = {{2,"a"}, {1,"b"}, {2,"c"}};
sort(v.begin(), v.end(), [](const auto& a, const auto& b){
    if (a.first != b.first) return a.first < b.first;
    return a.second < b.second;
});
```

### Sorting structs
```cpp
struct Person { string name; int age; };

vector<Person> ps = /*...*/;

// by age ascending, then name
sort(ps.begin(), ps.end(), [](const Person& a, const Person& b){
    return tie(a.age, a.name) < tie(b.age, b.name);
});
```

### Multi-key with mixed asc/desc
```cpp
// dept asc, score desc, name asc
sort(ps.begin(), ps.end(), [](const Person& a, const Person& b){
    if (a.dept != b.dept)   return a.dept < b.dept;
    if (a.score != b.score) return a.score > b.score;   // desc
    return a.name < b.name;
});
```

### Pre-built functors
```cpp
sort(v.begin(), v.end(), greater<int>());   // descending ints
sort(v.begin(), v.end(), greater<>());      // C++14 transparent — deduces type
```

---

## Comparator for `set` / `map`

The comparator is part of the **type**.

### Using `greater`
```cpp
set<int, greater<int>> desc = {1, 2, 3};   // iterates 3, 2, 1
map<string, int, greater<string>> m;
```

### Custom struct
```cpp
struct ByAbs {
    bool operator()(int a, int b) const { return abs(a) < abs(b); }
};
set<int, ByAbs> s = {-3, 1, -2};   // ordered by |x| → -2, -3 doesn't work, see note
// note: equal-magnitude elements collapse to one; set considers them equal
```

### Map with custom key
```cpp
struct Point { int x, y; };
struct PointLess {
    bool operator()(const Point& a, const Point& b) const {
        return tie(a.x, a.y) < tie(b.x, b.y);
    }
};
map<Point, int, PointLess> m;
m[{1, 2}] = 10;
```

### Equivalence in ordered containers
For ordered containers, "equal" means `!cmp(a,b) && !cmp(b,a)`. Two elements that compare equivalent are considered duplicates in `set`/`map`.

---

## Comparator for `priority_queue`

`priority_queue` is a **max-heap** by default — the comparator returns true when the first arg has **lower** priority.

### Min-heap with `greater`
```cpp
priority_queue<int, vector<int>, greater<int>> pq;
pq.push(3); pq.push(1); pq.push(2);
pq.top();   // 1
```

### Min-heap of pairs
```cpp
priority_queue<pair<int,int>,
               vector<pair<int,int>>,
               greater<pair<int,int>>> pq;
```

### Custom struct
```cpp
struct Task { int prio; string name; };

struct Cmp {
    bool operator()(const Task& a, const Task& b) const {
        return a.prio < b.prio;   // higher prio first (max-heap by prio)
    }
};
priority_queue<Task, vector<Task>, Cmp> q;
```

### Lambda comparator (verbose for PQ — can't deduce type)
```cpp
auto cmp = [](const Task& a, const Task& b){ return a.prio < b.prio; };
priority_queue<Task, vector<Task>, decltype(cmp)> q(cmp);
```
Better: write a named functor struct.

---

## Comparator for `lower_bound` / `upper_bound`

Must match the comparator the range was sorted with.
```cpp
sort(v.begin(), v.end(), greater<int>());
auto it = lower_bound(v.begin(), v.end(), 5, greater<int>());
```

---

## Comparator for `unordered_*`

You don't pass a comparator — you pass a **hash** and **equality** instead. See [04 Unordered](04_unordered_containers.md) and [21 Hashing](21_hashing.md).

---

## Common pitfalls

| Mistake | Fix |
|---|---|
| Returning `<=` instead of `<` | Use strict `<` |
| Sorting by `-key` to get descending (overflow risk) | Use `greater<>` or `b < a` |
| Lambda comparator inside `priority_queue` template params | Use a named struct |
| Different comparator for sort vs binary_search | Use the same one in both |
| Mutating a key inside `set`/`map` | Erase, mutate, re-insert |

## See also
- [03 Associative Containers](03_associative_containers.md)
- [05 Container Adapters](05_container_adapters.md)
- [10 Sorting & Search](10_algorithms_sorting_searching.md)
