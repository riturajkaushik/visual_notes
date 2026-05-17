# 01 — STL Foundations

> What the STL is, what you need to know before using it, and the header map.

## Setup
```cpp
#include <bits/stdc++.h>
using namespace std;
```

## The STL in one paragraph
- **Containers** hold data (`vector`, `map`, ...).
- **Iterators** are generalized pointers used to traverse containers.
- **Algorithms** operate on iterator ranges `[first, last)` — they don't know the container type.
- **Function objects** (functors, lambdas) plug behavior into algorithms.

The big idea: algorithms and containers are decoupled via iterators, so `sort` works on any random-access range.

## Required C++ Basics

### Function templates
```cpp
template <typename T>
T add(T a, T b) { return a + b; }

add(1, 2);          // T = int
add(1.5, 2.5);      // T = double
```

### Class templates
```cpp
template <typename T>
struct Box { T value; };

Box<int> b{42};
```

### Template type deduction with `auto`
```cpp
auto x = 42;            // int
auto y = 3.14;          // double
auto& r = x;            // int&
const auto& cr = x;     // const int&
```

### `decltype` and `decltype(auto)`
```cpp
int  i = 0;
int& r = i;
decltype(i) a = 1;      // int
decltype(r) b = i;      // int&
decltype(auto) c = r;   // int&   // C++14: preserves reference
```

### References and const references
```cpp
void by_value(vector<int> v);              // copies
void by_ref(vector<int>& v);               // can modify
void by_cref(const vector<int>& v);        // read-only, no copy — prefer this
```

### Move semantics
```cpp
vector<int> a(1000, 7);
vector<int> b = std::move(a);   // O(1), steals buffer
// a is now in a valid but unspecified state
```

### Lambdas
```cpp
auto sq = [](int x) { return x * x; };
sort(begin, end, [](int a, int b){ return a > b; });
```

### Operator overloading (for use in containers)
```cpp
struct P {
    int x, y;
    bool operator<(const P& o) const { return tie(x, y) < tie(o.x, o.y); }
};
set<P> s;  // works because operator< is defined
```

### RAII
Resources (memory, files, locks) are acquired in a constructor and released in a destructor. Every STL container is RAII.

### Copy vs move constructors
```cpp
struct S {
    S(const S&) { /* copy */ }
    S(S&&) noexcept { /* move */ }
};
```
`noexcept` move enables `vector` to move instead of copy on reallocation.

### Initializer lists
```cpp
vector<int> v = {1, 2, 3, 4};
map<string,int> m = {{"a",1}, {"b",2}};
```

## Header Cheat Sheet

| Header | Provides |
|---|---|
| `<vector>` | `vector` |
| `<array>` | `array` |
| `<deque>` | `deque` |
| `<list>` / `<forward_list>` | doubly/singly linked lists |
| `<set>` | `set`, `multiset` |
| `<map>` | `map`, `multimap` |
| `<unordered_set>` | `unordered_set`, `unordered_multiset` |
| `<unordered_map>` | `unordered_map`, `unordered_multimap` |
| `<queue>` | `queue`, `priority_queue` |
| `<stack>` | `stack` |
| `<algorithm>` | `sort`, `find`, `transform`, ... |
| `<numeric>` | `accumulate`, `iota`, `reduce` (C++17), `gcd`/`lcm` (C++17) |
| `<iterator>` | `back_inserter`, `istream_iterator`, ... |
| `<functional>` | `less`, `greater`, `function`, `bind`, ... |
| `<utility>` | `pair`, `move`, `swap`, `forward` |
| `<tuple>` | `tuple`, `tie`, `make_tuple` |
| `<memory>` | `unique_ptr`, `shared_ptr`, `make_unique` (C++14), `make_shared` |
| `<string>` | `string` |
| `<string_view>` | `string_view` (C++17) |
| `<bitset>` | `bitset` |
| `<chrono>` | clocks, durations, time points |
| `<random>` | engines, distributions |
| `<optional>` | `optional` (C++17) |
| `<variant>` | `variant` (C++17) |
| `<any>` | `any` (C++17) |
| `<filesystem>` | `filesystem::path`, ... (C++17) |
| `<execution>` | execution policies (C++17) |

**Convenience for CP**: `#include <bits/stdc++.h>` pulls everything (GCC/MinGW only — non-portable).

## See also
- [02 Sequence Containers](02_sequence_containers.md)
- [07 Iterators](07_iterators.md)
- [18 C++14 Features](18_cpp14_features.md) · [19 C++17 Features](19_cpp17_features.md)
