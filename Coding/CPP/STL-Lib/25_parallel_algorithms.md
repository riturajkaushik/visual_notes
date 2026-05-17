# 25 — Parallel Algorithms (C++17)

> Execution policies on standard algorithms via `<execution>`.

## Setup
```cpp
#include <bits/stdc++.h>
#include <execution>     // C++17
using namespace std;
```

**Build note:** GCC requires linking TBB: `g++ -std=c++17 file.cpp -ltbb`. Clang/libc++ likewise needs TBB on most platforms. MSVC supports them out of the box.

---

## The three (then four) execution policies

| Policy | Meaning |
|---|---|
| `execution::seq` | sequential — same as no policy |
| `execution::par` | may run in parallel on multiple threads |
| `execution::par_unseq` | may parallelize **and** vectorize (SIMD) |
| `execution::unseq` (C++20) | vectorize without threading |

---

## Basic usage

```cpp
vector<int> v(10'000'000);
iota(v.begin(), v.end(), 0);

sort(execution::par, v.begin(), v.end());
auto sum = reduce(execution::par, v.begin(), v.end(), 0LL);
for_each(execution::par, v.begin(), v.end(), [](int& x){ x *= 2; });
```

---

## Algorithms that commonly support policies

- `sort`, `stable_sort`, `partial_sort`
- `for_each`, `for_each_n`
- `transform`
- `reduce`, `transform_reduce`
- `inclusive_scan`, `exclusive_scan`, `transform_inclusive_scan`, `transform_exclusive_scan`
- `count`, `count_if`
- `find`, `find_if`, `find_if_not`, `find_end`, `find_first_of`
- `all_of`, `any_of`, `none_of`
- `copy`, `move`, `fill`, `generate`, `replace`
- `merge`, `inplace_merge`
- `partition`, `stable_partition`
- `equal`, `mismatch`
- `unique`, `unique_copy`
- `min_element`, `max_element`, `minmax_element`

---

## Side-effect rules

The body **must be safe to call concurrently** on independent elements.

```cpp
vector<int> v = /*...*/;
int sum = 0;
// BAD: race on `sum`
for_each(execution::par, v.begin(), v.end(),
         [&](int x){ sum += x; });

// GOOD: use reduction
auto s = reduce(execution::par, v.begin(), v.end(), 0);
```

`reduce` and the scan family explicitly *may* reorder operations. The combining function must be **associative**; with `par_unseq` it should also be **commutative**.

---

## Floating-point caveat

Floating-point addition is **not associative**:
```cpp
(a + b) + c   !=   a + (b + c)    // in general
```
So a parallel `reduce` over `double`s may produce slightly different sums on different runs. If you need bitwise-stable sums, use `accumulate` (sequential).

---

## Exception behavior

A throwing element op under a parallel policy terminates the program via `std::terminate` instead of propagating — by design. Catch inside the lambda if you need to keep going.

```cpp
for_each(execution::par, v.begin(), v.end(),
         [](Item& it){
             try { it.process(); }
             catch (...) { /* swallow or record */ }
         });
```

---

## When parallel helps (and when it doesn't)

Worth it:
- Large n (≥ 10^5) and non-trivial per-element work.
- CPU-bound element ops with no shared state.
- Sorting many elements.

Not worth it:
- Small n — thread setup dominates.
- Trivial work (e.g. `transform` that just multiplies).
- Memory-bound code without enough compute per element.
- Anything that touches shared state without atomics or reductions.

---

## Compiler/library support reality check (as of writing)

- **MSVC**: full support.
- **libstdc++ (g++)**: implements via Intel TBB. Without `-ltbb` the parallel policies fall back to sequential.
- **libc++ (clang)**: requires TBB; varies by platform.

If parallel feels slow, check whether your binary actually links a parallel backend.

## See also
- [13 Numeric Algorithms](13_numeric_algorithms.md) — `reduce`, scans
- [19 C++17 Features](19_cpp17_features.md)
