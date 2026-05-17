# 14 — Lambdas & Function Objects

> Lambdas, standard functors from `<functional>`, `std::function`, `std::bind`.

## Setup
```cpp
#include <bits/stdc++.h>
using namespace std;
```

---

## Lambda syntax

```cpp
auto f = [capture](params) -> return_type { body; };
```

Most pieces are optional:
```cpp
auto g = []{ return 42; };           // no capture, no params, deduced return
auto h = [](int a, int b){ return a + b; };
```

## Captures

```cpp
int x = 10, y = 20;

auto by_val   = [x, y]{ return x + y; };          // copies
auto by_ref   = [&x, &y]{ x++; y++; };            // references
auto cap_all_val = [=]{ return x + y; };          // all by value
auto cap_all_ref = [&]{ x++; };                   // all by reference
auto mixed   = [=, &y]{ y += x; };                // x by value, y by ref
```

### `mutable` — allow modifying captured-by-value
```cpp
int count = 0;
auto tick = [count]() mutable { return ++count; };
tick(); tick(); tick();   // returns 1, 2, 3
// outer 'count' still 0 — the lambda has its own copy
```

### Init capture (C++14)
```cpp
auto p = make_unique<int>(42);
auto holder = [up = std::move(p)]() {
    return *up;
};
```
Lets you capture move-only types and rename captures.

### Generic lambdas (C++14)
```cpp
auto add = [](auto a, auto b) { return a + b; };
add(1, 2);        // int + int
add(1.5, 2.5);    // double + double
add(string("hi "), string("there"));   // string concat
```

---

## Common lambda + algorithm combos

```cpp
vector<int> v = {3, 1, 4, 1, 5, 9, 2, 6};

sort(v.begin(), v.end(), [](int a, int b){ return a > b; });

auto it = find_if(v.begin(), v.end(), [](int x){ return x > 5; });

int cnt = count_if(v.begin(), v.end(), [](int x){ return x % 2 == 0; });

vector<int> sq(v.size());
transform(v.begin(), v.end(), sq.begin(), [](int x){ return x*x; });
```

---

## Standard function objects (`<functional>`)

### Comparators
```cpp
sort(v.begin(), v.end(), less<int>());       // ascending (default)
sort(v.begin(), v.end(), greater<int>());    // descending
priority_queue<int, vector<int>, greater<int>> minHeap;
```

Other comparators: `less_equal`, `greater_equal`, `equal_to`, `not_equal_to`.

### Arithmetic
```cpp
plus<int>()(2, 3);          // 5
minus<int>()(5, 2);         // 3
multiplies<int>()(3, 4);    // 12
divides<int>()(10, 2);      // 5
modulus<int>()(10, 3);      // 1
negate<int>()(7);           // -7
```

### Logical / bitwise
```cpp
logical_and<bool>()(true, false);   // false
logical_or<bool>()(true, false);    // true
logical_not<bool>()(true);          // false

bit_and<int>()(0b1100, 0b1010);     // 0b1000
bit_or<int>() (0b1100, 0b1010);     // 0b1110
bit_xor<int>()(0b1100, 0b1010);     // 0b0110
```

### Used in transforms/reductions
```cpp
vector<int> a = {1,2,3,4};
int s = accumulate(a.begin(), a.end(), 1, multiplies<int>());     // 24

vector<int> a2(4), b2(4);
transform(a.begin(), a.end(), b2.begin(), negate<int>());          // {-1,-2,-3,-4}
```

### Transparent operators (C++14)
```cpp
sort(v.begin(), v.end(), greater<>());          // empty <> — deduces type
plus<>()(1.5, 2);                                // mixes int+double
```

---

## `std::function` — type-erased callable

```cpp
function<int(int,int)> f;
f = [](int a, int b){ return a + b; };
f(3, 4);   // 7

f = multiplies<int>();
f(3, 4);   // 12

int free_add(int a, int b) { return a + b; }
f = &free_add;

struct Functor { int operator()(int a, int b) const { return a - b; } };
f = Functor();
```

**When to use:** storing heterogeneous callables (callbacks, event handlers, visitor tables).
**Cost:** small heap allocation possible, virtual-call-like dispatch. Prefer `auto` for plain lambdas; reach for `function` only when the type really must be erased.

```cpp
map<string, function<int(int,int)>> ops = {
    {"+", plus<>()},
    {"-", minus<>()},
    {"*", multiplies<>()},
};
ops["*"](3, 4);   // 12
```

---

## `std::bind` and placeholders

```cpp
using namespace std::placeholders;

int sub(int a, int b) { return a - b; }
auto sub_from_10 = bind(sub, 10, _1);
sub_from_10(3);                              // 7

auto swap_args = bind(sub, _2, _1);
swap_args(3, 10);                            // 7

// Bind a member function
struct Counter { int n=0; void inc(int by){ n += by; } };
Counter c;
auto inc5 = bind(&Counter::inc, &c, 5);
inc5(); inc5();                              // c.n == 10
```

**Modern advice:** prefer lambdas. They're clearer, often faster, and handle moves better.
```cpp
auto sub_from_10 = [](int x){ return 10 - x; };   // simpler than bind
```

## See also
- [20 Comparators](20_comparators.md)
- [18 C++14 Features](18_cpp14_features.md) — init capture, generic lambdas
