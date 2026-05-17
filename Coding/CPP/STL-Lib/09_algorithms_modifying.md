# 09 — Modifying Algorithms

> Algorithms that write into a range. From `<algorithm>`.

## Setup
```cpp
#include <bits/stdc++.h>
using namespace std;
```

---

## Copy variants

```cpp
vector<int> src = {1, 2, 3, 4, 5};
vector<int> dst(5);
copy(src.begin(), src.end(), dst.begin());

// growing destination
vector<int> dst2;
copy(src.begin(), src.end(), back_inserter(dst2));   // dst2 = {1,2,3,4,5}
```

### `copy_if`
```cpp
vector<int> evens;
copy_if(src.begin(), src.end(), back_inserter(evens),
        [](int x){ return x % 2 == 0; });
// evens = {2, 4}
```

### `copy_n`
```cpp
vector<int> first3(3);
copy_n(src.begin(), 3, first3.begin());   // {1,2,3}
```

### `copy_backward`
```cpp
vector<int> v = {1, 2, 3, 4, 5, 0, 0};
copy_backward(v.begin(), v.begin() + 5, v.end());
// v = {1, 2, 1, 2, 3, 4, 5}  — used to shift right safely
```

---

## Move variants
Same shape as copy, but moves elements (leaves source in valid-but-unspecified state).
```cpp
vector<string> a = {"x", "y", "z"};
vector<string> b(3);
move(a.begin(), a.end(), b.begin());     // a's strings now moved into b

move_backward(a.begin(), a.end(), some_end);
```

---

## `fill`, `fill_n`

```cpp
vector<int> v(5);
fill(v.begin(), v.end(), 7);   // {7,7,7,7,7}

fill_n(v.begin(), 3, -1);      // {-1,-1,-1,7,7}
```

---

## `transform` — map a function over a range

```cpp
vector<int> v = {1, 2, 3, 4};
vector<int> sq(v.size());
transform(v.begin(), v.end(), sq.begin(),
          [](int x){ return x * x; });
// sq = {1, 4, 9, 16}

// Two-range form
vector<int> a = {1, 2, 3}, b = {10, 20, 30}, sums(3);
transform(a.begin(), a.end(), b.begin(), sums.begin(),
          [](int x, int y){ return x + y; });
// sums = {11, 22, 33}
```

---

## `generate`, `generate_n` — fill via a callable

```cpp
vector<int> v(5);
int i = 0;
generate(v.begin(), v.end(), [&]{ return i++; });    // {0,1,2,3,4}

vector<int> r(5);
generate_n(r.begin(), 5, [n=10]() mutable { return n++; });
// r = {10,11,12,13,14}
```

---

## `remove`, `remove_if` — partition, then erase

`remove` does **not** change container size. It moves "kept" elements to the front and returns the new logical end.

```cpp
vector<int> v = {1, 2, 3, 2, 4, 2};
auto new_end = remove(v.begin(), v.end(), 2);
// v = {1, 3, 4, ?, ?, ?}, new_end points past 4
v.erase(new_end, v.end());     // erase-remove idiom
// v = {1, 3, 4}

vector<int> w = {1, 2, 3, 4, 5};
w.erase(remove_if(w.begin(), w.end(),
                  [](int x){ return x > 3; }),
        w.end());
// w = {1, 2, 3}
```
**Gotcha:** `remove` alone leaves the container size unchanged. You almost always want `container.erase(remove(...), container.end())`.

---

## `replace`, `replace_if`

```cpp
vector<int> v = {1, 2, 3, 2, 1};
replace(v.begin(), v.end(), 2, 99);                       // {1,99,3,99,1}
replace_if(v.begin(), v.end(),
           [](int x){ return x > 50; }, 0);               // {1,0,3,0,1}
```

---

## `swap`, `swap_ranges`

```cpp
int a = 1, b = 2;
swap(a, b);

vector<int> v1 = {1, 2, 3}, v2 = {7, 8, 9};
swap_ranges(v1.begin(), v1.end(), v2.begin());
// v1 = {7,8,9}, v2 = {1,2,3}

vector<int> A(1000), B(1000);
swap(A, B);   // O(1) for vectors — swaps internal pointers
```

---

## `reverse`

```cpp
vector<int> v = {1, 2, 3, 4};
reverse(v.begin(), v.end());     // {4, 3, 2, 1}

string s = "hello";
reverse(s.begin(), s.end());     // "olleh"
```

---

## `rotate`

```cpp
vector<int> v = {1, 2, 3, 4, 5};
rotate(v.begin(), v.begin() + 2, v.end());
// v = {3, 4, 5, 1, 2}
```
`rotate(first, middle, last)`: makes `middle` the new first element.

---

## `shuffle` (and the dead `random_shuffle`)

`random_shuffle` is **deprecated in C++14, removed in C++17**. Use `shuffle` with a real engine.

```cpp
vector<int> v = {1,2,3,4,5};
mt19937 rng(random_device{}());
shuffle(v.begin(), v.end(), rng);
```

---

## `unique` — collapse consecutive duplicates

```cpp
vector<int> v = {1, 1, 2, 3, 3, 3, 4};
auto e = unique(v.begin(), v.end());     // v = {1,2,3,4,?,?,?}
v.erase(e, v.end());                     // v = {1,2,3,4}
```
**Only removes consecutive duplicates** — sort first to dedupe fully:
```cpp
sort(v.begin(), v.end());
v.erase(unique(v.begin(), v.end()), v.end());
```

---

## Erase-remove idiom (the most common one)

```cpp
v.erase(remove(v.begin(), v.end(), x), v.end());                 // remove all x
v.erase(remove_if(v.begin(), v.end(), pred), v.end());           // remove all matches
v.erase(unique(v.begin(), v.end()), v.end());                    // dedup consecutive
```

C++20 introduces `std::erase`/`std::erase_if` to wrap this, but in C++14/17 you write it explicitly.

## See also
- [08 Non-Modifying](08_algorithms_non_modifying.md)
- [22 Idioms](22_idioms.md)
