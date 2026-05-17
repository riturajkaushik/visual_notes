# 13 — Numeric Algorithms

> From `<numeric>`. Reductions, scans, sequence generation, plus C++17 additions.

## Setup
```cpp
#include <bits/stdc++.h>
using namespace std;
```

---

## `accumulate` — sum / fold

```cpp
vector<int> v = {1, 2, 3, 4};
int s = accumulate(v.begin(), v.end(), 0);          // 10
```

### Overflow trap
```cpp
vector<int> big(1'000'000, 1'000'000);
int    bad = accumulate(big.begin(), big.end(), 0);     // overflow!
long long ok = accumulate(big.begin(), big.end(), 0LL); // correct
```
**Rule:** the accumulator type comes from the initial value. Use `0LL` for `long long`, `0.0` for `double`.

### Custom binary op (fold)
```cpp
vector<int> v = {1, 2, 3, 4};
int prod = accumulate(v.begin(), v.end(), 1, multiplies<int>());   // 24
string  s = accumulate(v.begin(), v.end(), string(),
                       [](string acc, int x){
                           return acc + to_string(x);
                       });
// s = "1234"
```

---

## `inner_product` — dot product / generalized

```cpp
vector<int> a = {1, 2, 3};
vector<int> b = {4, 5, 6};
int dot = inner_product(a.begin(), a.end(), b.begin(), 0);
// 1*4 + 2*5 + 3*6 = 32

// Custom ops: e.g., max of products, sum of (a==b)
int eq = inner_product(a.begin(), a.end(), b.begin(), 0,
                       plus<int>(),
                       [](int x, int y){ return x == y ? 1 : 0; });
```

---

## `partial_sum` — prefix-sum (inclusive)

```cpp
vector<int> v = {1, 2, 3, 4};
vector<int> ps(v.size());
partial_sum(v.begin(), v.end(), ps.begin());
// ps = {1, 3, 6, 10}

// custom op (e.g., prefix max)
partial_sum(v.begin(), v.end(), ps.begin(),
            [](int a, int b){ return max(a, b); });
```

For a typical `pref[i+1] = pref[i] + a[i]` style sized n+1, write the loop yourself — see [22 Idioms](22_idioms.md).

---

## `adjacent_difference` — pairwise differences

```cpp
vector<int> v = {1, 3, 6, 10};
vector<int> d(v.size());
adjacent_difference(v.begin(), v.end(), d.begin());
// d = {1, 2, 3, 4}   (first element copied as-is)
```
Inverse of `partial_sum`.

---

## `iota` — fill with increasing sequence

```cpp
vector<int> v(5);
iota(v.begin(), v.end(), 1);            // {1,2,3,4,5}
iota(v.begin(), v.end(), 10);           // {10,11,12,13,14}
```

### Common pattern: indices sorted by value
```cpp
vector<int> a = {30, 10, 20};
vector<int> idx(a.size());
iota(idx.begin(), idx.end(), 0);
sort(idx.begin(), idx.end(),
     [&](int i, int j){ return a[i] < a[j]; });
// idx = {1, 2, 0}   — indices of sorted-by-value
```

---

# C++17 additions

## `reduce` — parallel-friendly sum

```cpp
vector<int> v = {1, 2, 3, 4};
int s = reduce(v.begin(), v.end());            // 10 (default init = T{})
int t = reduce(v.begin(), v.end(), 0);
int p = reduce(v.begin(), v.end(), 1, multiplies<int>());
```

**Difference from `accumulate`:**
- `accumulate` applies left-to-right, sequential.
- `reduce` may **reorder** operations. The binary op must be **associative** and **commutative** to give deterministic results — fine for int +/*, sketchy for floats and matrix mul.
- `reduce` works with execution policies (see [25](25_parallel_algorithms.md)).

```cpp
#include <execution>
auto s = reduce(execution::par, v.begin(), v.end(), 0);
```

## `transform_reduce` — map+reduce in one pass

```cpp
vector<int> a = {1, 2, 3}, b = {4, 5, 6};

// dot product (sum of products) — like inner_product but unordered
int dot = transform_reduce(a.begin(), a.end(), b.begin(), 0);  // 32

// single-range form: sum of squares
int sq = transform_reduce(a.begin(), a.end(), 0,
                          plus<int>(),
                          [](int x){ return x*x; });
// 1 + 4 + 9 = 14
```

## `inclusive_scan` / `exclusive_scan` — prefix-sum, parallel-friendly

```cpp
vector<int> v = {1, 2, 3, 4};
vector<int> inc(4), exc(4);

inclusive_scan(v.begin(), v.end(), inc.begin());
// inc = {1, 3, 6, 10}   (like partial_sum, but may reorder)

exclusive_scan(v.begin(), v.end(), exc.begin(), 0);
// exc = {0, 1, 3, 6}    (initial value at front; last element of v not included)
```

## `transform_inclusive_scan` / `transform_exclusive_scan`

```cpp
vector<int> v = {1, 2, 3, 4};
vector<int> out(4);
transform_inclusive_scan(v.begin(), v.end(), out.begin(),
                         plus<int>(),
                         [](int x){ return x*x; });
// scans of squares: {1, 5, 14, 30}
```

## `gcd`, `lcm`

```cpp
gcd(12, 18);    // 6
lcm(4, 6);      // 12
gcd(0, 5);      // 5
```
Work on any integral types; return type is the common type.

---

## Quick mapping

| Task | C++14 | C++17 |
|---|---|---|
| Sum | `accumulate(..., 0)` | `reduce(...)` |
| Sum of f(x) | `accumulate(..., 0, [&](a,x){...})` | `transform_reduce(...)` |
| Dot product | `inner_product` | `transform_reduce(a,a+n,b,0)` |
| Prefix sums | `partial_sum` | `inclusive_scan` / `exclusive_scan` |
| Differences | `adjacent_difference` | same |
| Generate 0..n-1 | `iota` | same |
| GCD/LCM | (manual) | `gcd`, `lcm` |

## See also
- [22 Idioms](22_idioms.md) — prefix sum, coordinate compression
- [25 Parallel Algorithms](25_parallel_algorithms.md)
