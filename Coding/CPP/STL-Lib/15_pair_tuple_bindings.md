# 15 — Pair, Tuple & Structured Bindings

> `std::pair`, `std::tuple`, and C++17 structured bindings.

## Setup
```cpp
#include <bits/stdc++.h>
using namespace std;
```

---

## `std::pair`

```cpp
pair<int, string> p = {1, "one"};
p.first;     // 1
p.second;    // "one"

auto p2 = make_pair(2, string("two"));    // type deduced
auto p3 = pair{3, string("three")};       // C++17 CTAD
```

### Comparison (lex)
```cpp
pair<int,int> a = {1, 2}, b = {1, 3};
a < b;   // true  (1==1, 2<3)
```

### Use as map / set element
```cpp
set<pair<int,int>> s = {{1,2},{3,4}};
map<pair<int,int>, int> grid;
grid[{0,0}] = 5;
```

### Returning two values
```cpp
pair<int, int> minmax_pair(const vector<int>& v) {
    auto [lo, hi] = minmax_element(v.begin(), v.end());
    return {*lo, *hi};
}
```

---

## `std::tuple`

```cpp
tuple<int, string, double> t = {1, "abc", 3.14};
auto t2 = make_tuple(2, string("xyz"), 1.5);
auto t3 = tuple{3, string("hi"), 2.71};   // C++17 CTAD
```

### Access by index
```cpp
get<0>(t);   // 1
get<1>(t);   // "abc"
get<2>(t);   // 3.14
get<string>(t);   // "abc" — by type (must be unique in the tuple)
```

### `tie` — unpack into existing variables
```cpp
int a; string s; double d;
tie(a, s, d) = t;

// ignore some
int x; string y;
tie(x, ignore, y) = make_tuple(1, 3.14, "hi");
// y = "hi"
```

### `tie` for multi-key compare
```cpp
struct Row { int dept; int score; string name; };
bool less_row(const Row& a, const Row& b) {
    return tie(a.dept, a.score, a.name)
         < tie(b.dept, b.score, b.name);
}
```

### Concatenate tuples
```cpp
auto big = tuple_cat(make_tuple(1, 2), make_tuple("x", "y"));
// tuple<int,int,const char*,const char*>
```

### Size / element-type queries
```cpp
constexpr auto N = tuple_size<decltype(t)>::value;       // 3
using First = tuple_element<0, decltype(t)>::type;       // int
```

---

## Structured Bindings (C++17)

A single statement that unpacks pairs, tuples, arrays, or aggregate structs.

### With pair / tuple
```cpp
auto [a, b]    = pair{1, 2};
auto [x, y, z] = tuple{10, "hi", 3.14};
```

### With map (the killer use case)
```cpp
map<string, int> m = {{"a",1},{"b",2}};
for (auto& [key, value] : m) {
    cout << key << "->" << value << ' ';
}
```

### With `insert` / `emplace` return value
```cpp
map<int,int> m;
auto [it, inserted] = m.insert({1, 100});
if (!inserted) cout << "key 1 already present";
```

### With arrays
```cpp
int arr[3] = {10, 20, 30};
auto [a, b, c] = arr;
```

### With structs
```cpp
struct Point { int x, y, z; };
Point p{1, 2, 3};
auto [x, y, z] = p;
auto& [rx, ry, rz] = p;
rx = 99;                       // p.x is now 99
```
**Requirements:** struct must be an "aggregate" — no user-declared constructors, all public members, no virtual.

### Combining with multi-return functions
```cpp
auto stats(const vector<int>& v) {
    int sum = 0, mx = INT_MIN;
    for (int x : v) { sum += x; mx = max(mx, x); }
    return tuple{sum, mx, (double)sum / v.size()};
}

auto [sum, mx, avg] = stats({1,2,3,4});
```

### `const auto&` / `auto&` for efficiency
```cpp
vector<pair<string,int>> kv = /*...*/;
for (const auto& [k, v] : kv) cout << k << '=' << v << ' ';  // no copies
```

---

## When to use which

| Need | Use |
|---|---|
| Two values, named or not | `pair` |
| 3+ values, ad-hoc | `tuple` |
| 3+ named values, reused | a named `struct` |
| Multi-key comparison | `tie` with `<` |
| Unpacking | structured bindings (C++17) |

## See also
- [14 Lambdas](14_function_objects_lambdas.md)
- [19 C++17 Features](19_cpp17_features.md)
- [20 Comparators](20_comparators.md) — `tie` for multi-key sort
