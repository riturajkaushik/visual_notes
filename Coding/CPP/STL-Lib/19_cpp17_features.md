# 19 — C++17 STL-Relevant Features

> What C++17 added that matters for STL usage.

## Setup
```cpp
#include <bits/stdc++.h>
using namespace std;
```

---

## Structured bindings

```cpp
auto [a, b] = pair{1, 2};
auto [x, y, z] = tuple{10, "hi", 3.14};

map<string,int> m = {{"a",1},{"b",2}};
for (auto& [k, v] : m) cout << k << '=' << v << ' ';

struct P { int x, y; };
auto [px, py] = P{3, 4};
```
See [15 Pair/Tuple](15_pair_tuple_bindings.md).

---

## `if constexpr` — compile-time branching

```cpp
template <typename T>
string describe(T x) {
    if constexpr (is_integral_v<T>)         return "integer";
    else if constexpr (is_floating_point_v<T>) return "float";
    else                                       return "other";
}
describe(1);      // "integer"
describe(1.0);    // "float"
describe("hi");   // "other"
```

The "dead" branches aren't compiled, so you can use code that would otherwise be a hard error:
```cpp
template <typename T>
void print(T v) {
    if constexpr (is_pointer_v<T>) cout << *v;   // only compiled when T is a pointer
    else                            cout << v;
}
```

---

## Class Template Argument Deduction (CTAD)

```cpp
vector v = {1, 2, 3};                 // vector<int>
pair  p = {1, 3.14};                  // pair<int, double>
tuple t = {1, "hi", 3.14};            // tuple<int, const char*, double>
array a = {1, 2, 3};                  // array<int, 3>

// Custom: deduction guides
template <typename T> struct Box { T v; };
template <typename T> Box(T) -> Box<T>;
Box b{42};            // Box<int>
```

---

## Inline variables

Headers can define variables without ODR violations.

```cpp
// in a header
inline constexpr int kMaxLen = 1024;
inline string kBanner = "Hello";
```
Useful for header-only libs and shared constants.

---

## Fold expressions (parameter pack folding)

```cpp
template <typename... Args>
auto sum(Args... xs) {
    return (xs + ...);                  // unary right fold
}
sum(1, 2, 3, 4);                        // 10

template <typename... Args>
void print(Args... xs) {
    ((cout << xs << ' '), ...);         // comma fold
}
print(1, "hi", 3.14);                   // 1 hi 3.14
```

---

## `std::optional`, `std::variant`, `std::any`
See [17 Utility Components](17_utility_components.md).

---

## `std::string_view`
See [06 Strings](06_strings.md).

---

## `std::filesystem`
See [17 Utility Components](17_utility_components.md).

---

## Parallel algorithms
See [25 Parallel Algorithms](25_parallel_algorithms.md).

---

## New numeric algorithms
- `reduce`, `transform_reduce`, `inclusive_scan`, `exclusive_scan`, `gcd`, `lcm` — see [13 Numeric](13_numeric_algorithms.md).

---

## `std::clamp`
```cpp
clamp(5, 0, 10);     // 5
clamp(-3, 0, 10);    // 0
clamp(99, 0, 10);    // 10
```

---

## `std::byte`

A strongly-typed byte for raw memory work (not `char`, not arithmetic).
```cpp
byte b{0x42};
byte c = b << 1;
unsigned u = to_integer<unsigned>(b);
```

---

## Type-trait `_v` and `_t` helpers (some in C++14, broadened in C++17)

```cpp
is_integral_v<T>          // shorter than is_integral<T>::value
is_same_v<T, U>
remove_reference_t<T>     // shorter than typename remove_reference<T>::type
decay_t<T>
enable_if_t<cond, T>
```

---

## `if` / `switch` with initializer

```cpp
if (auto it = m.find(k); it != m.end()) {
    use(it->second);
}

switch (auto r = parse(); r.kind) {
    case Kind::Int:    handle(r.i); break;
    case Kind::String: handle(r.s); break;
}
```

---

## `std::invoke`

Uniform call syntax for callables, member functions, and member pointers.
```cpp
invoke([](int x){ return x*x; }, 5);     // 25

struct S { int val = 10; int times(int n){ return val * n; } };
S s;
invoke(&S::times, s, 3);                  // 30
invoke(&S::val, s);                       // 10
```

---

## `std::apply` — call a function with tuple args

```cpp
auto t = make_tuple(1, 2, 3);
auto sum = apply([](int a, int b, int c){ return a + b + c; }, t);   // 6
```

---

## Nested namespaces

```cpp
namespace a::b::c {
    int f() { return 42; }
}
// instead of: namespace a { namespace b { namespace c { ... }}}
```

---

## `std::not_fn`

Negates a callable. Replaces the deprecated `not1` / `not2`.
```cpp
auto is_even = [](int x){ return x % 2 == 0; };
auto is_odd  = not_fn(is_even);
count_if(v.begin(), v.end(), is_odd);
```

---

## `std::sample` — random subset

```cpp
vector<int> v(100);
iota(v.begin(), v.end(), 0);
vector<int> picked;
mt19937 rng(random_device{}());
sample(v.begin(), v.end(), back_inserter(picked), 10, rng);
// picked has 10 distinct values from v
```

## See also
- [18 C++14 Features](18_cpp14_features.md)
- [17 Utility Components](17_utility_components.md)
