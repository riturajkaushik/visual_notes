# 27 — Debugging STL Code

> Common mistakes and how to catch them.

## Setup
```cpp
#include <bits/stdc++.h>
using namespace std;
```

---

## Off-by-one with `size()`

`size()` returns `size_t` (unsigned). Comparing a signed `i` against it can subtract through zero.
```cpp
for (int i = 0; i <= v.size(); i++)   // BUG: extra iteration; also signed/unsigned warning
    cout << v[i];

// Fix
for (size_t i = 0; i < v.size(); i++)
    cout << v[i];

// Or
for (int i = 0; i < (int)v.size(); i++)
    cout << v[i];

// Or iterator / range-for
for (auto x : v) cout << x;
```

`size_t` underflow:
```cpp
vector<int> v;
for (size_t i = 0; i < v.size() - 1; i++)  // BUG when v.size() == 0
    ;
```

---

## Dereferencing `end()`

`end()` is **one past the last element**. Reading from it is UB.
```cpp
vector<int> v = {1, 2, 3};
auto it = v.end();
cout << *it;        // UB
cout << *(it - 1);  // 3
```

---

## Iterator invalidation

```cpp
vector<int> v = {1, 2, 3};
for (auto it = v.begin(); it != v.end(); ++it) {
    if (*it == 2) v.erase(it);     // BUG: it invalidated, then ++it
}
```
Fix — use `erase`'s return value, or `remove_if`:
```cpp
for (auto it = v.begin(); it != v.end(); ) {
    if (*it == 2) it = v.erase(it);
    else          ++it;
}
// Or
v.erase(remove(v.begin(), v.end(), 2), v.end());
```

Same trap with `vector::push_back` invalidating saved iterators if reallocation happens.

---

## `unordered_map::operator[]` inserting silently

```cpp
unordered_map<string,int> m = {{"a",1}};
if (m["missing"] == 5) {}      // inserts "missing" -> 0, then compares
```
Use `find` or `count` to peek.

---

## `vector<bool>` is not a normal vector

It's a packed bitset proxy. `vector<bool>::reference` is not `bool&`.
```cpp
vector<bool> v = {true, false};
auto& b = v[0];          // 'auto&' is a proxy, NOT bool&
bool* p = &v[0];         // compile error
```
Prefer `vector<char>` or `bitset<N>` when you need real references / pointers.

---

## `auto` copying when you wanted a reference

```cpp
map<string, vector<int>> m = /*...*/;
for (auto kv : m) { kv.second.push_back(1); }   // BUG: modifying a copy
for (auto& kv : m) { kv.second.push_back(1); }  // fix
```

---

## `accumulate` integer overflow

```cpp
vector<int> v(1'000'000, 1'000'000);
int sum = accumulate(v.begin(), v.end(), 0);      // overflow
long long ok = accumulate(v.begin(), v.end(), 0LL);
```

---

## Strict weak ordering violations

```cpp
sort(v.begin(), v.end(), [](int a, int b){ return a <= b; });   // BUG
// Use < strictly:
sort(v.begin(), v.end(), [](int a, int b){ return a < b; });
```
Symptoms: random crashes, hangs, or assertion failures on libcxx debug builds.

---

## Lifetime issues with `string_view`

```cpp
string_view view_of(string s)  { return s; }   // BUG: returns view to local
auto sv = view_of("hi");
cout << sv;                                    // UB
```

---

## Mutating keys inside `set` / `map`

Keys are stored as `const`. The library exposes them as const through iterators for a reason — mutating a key breaks the tree.
```cpp
set<Point> s;
for (auto& p : s) p.x++;   // would be UB if compiled — iterator yields const Point
// Correct flow: erase, mutate, re-insert.
```

---

## Forgetting to bounds-check

Use `at()` for development:
```cpp
v.at(i);                  // throws std::out_of_range
m.at("key");              // throws if missing
```
Then switch to `[]` once you're confident.

---

## Mixing `cin` with `scanf`

`ios::sync_with_stdio(false)` decouples them. After that, mixing is UB-flavored.

---

## Debug-mode containers

- **libstdc++ (g++)**: compile with `-D_GLIBCXX_DEBUG` to enable iterator and bounds checks.
- **libc++ (clang)**: compile with `-D_LIBCPP_DEBUG=1`.
- **MSVC**: debug iterators are on by default in Debug builds.

These catch most of the bugs above at runtime with clear diagnostics. Cost is significant — leave them on for tests/debug, off for release.

---

## Printing containers (cheap helper)

```cpp
template <typename C>
void print(const C& c) {
    cout << '[';
    bool first = true;
    for (const auto& x : c) {
        if (!first) cout << ", ";
        cout << x;
        first = false;
    }
    cout << "]\n";
}
print(vector<int>{1,2,3});       // [1, 2, 3]
print(set<int>{3,1,2});          // [1, 2, 3]
```

For maps:
```cpp
template <typename K, typename V>
void print(const map<K,V>& m) {
    cout << '{';
    bool first = true;
    for (const auto& [k, v] : m) {
        if (!first) cout << ", ";
        cout << k << ':' << v;
        first = false;
    }
    cout << "}\n";
}
```

---

## Sanitizers (highly recommended)

```bash
g++ -std=c++17 -O1 -g -fsanitize=address,undefined file.cpp
```
Catches out-of-bounds, use-after-free, signed overflow, and most of the UB above at runtime.

## See also
- [07 Iterators](07_iterators.md) — invalidation rules
- [24 Complexity](24_complexity.md)
