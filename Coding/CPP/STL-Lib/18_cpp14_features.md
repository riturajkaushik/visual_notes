# 18 — C++14 STL-Relevant Features

> What C++14 added that matters for STL usage.

## Setup
```cpp
#include <bits/stdc++.h>
using namespace std;
```

---

## Generic lambdas

`auto` in lambda parameters → template-like behavior.

```cpp
auto add = [](auto a, auto b){ return a + b; };
add(1, 2);                  // int
add(1.5, 2.5);              // double
add(string("hi "), string("there"));

// In algorithms
vector<pair<int,int>> v = /*...*/;
sort(v.begin(), v.end(), [](const auto& a, const auto& b){
    return a.second < b.second;
});
```

---

## `auto` return type deduction

```cpp
auto square(int x) { return x * x; }     // returns int

// Recursive functions need a return type or trailing return
auto factorial(int n) -> int {
    return n <= 1 ? 1 : n * factorial(n - 1);
}
```

---

## `decltype(auto)`

Use when you want to perfectly forward the return expression's category (value vs reference).

```cpp
int  x = 10;
int& get_ref()  { return x; }

auto a = get_ref();              // int (copy)
decltype(auto) b = get_ref();    // int& (reference preserved)

template <typename C, typename K>
decltype(auto) at_key(C& c, K&& k) {
    return c[forward<K>(k)];     // returns reference for map, etc.
}
```

---

## `std::make_unique`

C++11 had `make_shared` but **not** `make_unique`. C++14 fixes that.

```cpp
auto p = make_unique<int>(42);
auto arr = make_unique<int[]>(100);   // zero-initialized array

// Why prefer it over `new`:
// 1) No risk of leak from exceptions in argument evaluation
// 2) Symmetric with make_shared
// 3) No bare `new` in modern code
```
See [16 Smart Pointers](16_smart_pointers.md).

---

## Relaxed `constexpr`

`constexpr` functions can have loops, branches, mutations of local variables.

```cpp
constexpr int factorial(int n) {
    int r = 1;
    for (int i = 2; i <= n; ++i) r *= i;
    return r;
}

constexpr int F5 = factorial(5);   // 120, computed at compile time
static_assert(F5 == 120);
```

---

## Variable templates

```cpp
template <typename T>
constexpr T pi = T(3.1415926535897932385);

double d = pi<double>;
float  f = pi<float>;
```

Used widely in C++17 type trait helpers:
```cpp
template <typename T>
constexpr bool is_integral_v = is_integral<T>::value;   // standard in C++17
```

---

## Binary literals & digit separators

```cpp
int mask = 0b1100'1010;            // binary literal, ' separates digits
long big  = 1'000'000'000;
double pi = 3.141'592'653;
```

---

## `std::quoted` (in `<iomanip>`, since C++14)

```cpp
string s = R"(he said "hi")";
cout << quoted(s);             // "he said \"hi\""
stringstream ss;
ss << quoted(s);
string parsed;
ss >> quoted(parsed);          // parsed == s
```

---

## `std::exchange`

```cpp
int x = 1;
int old = exchange(x, 2);      // old=1, x=2

// Move-and-reset idiom
struct Move {
    unique_ptr<int> p;
    Move(Move&& other) noexcept
        : p(exchange(other.p, nullptr)) {}
};
```

---

## Heterogeneous lookup for ordered containers

Custom comparator with `is_transparent` enables lookups by a type different from the key.

```cpp
struct LessStr {
    using is_transparent = void;
    bool operator()(const string& a, const string& b) const { return a < b; }
    bool operator()(const string& a, string_view b)   const { return a < b; }
    bool operator()(string_view a,   const string& b) const { return a < b; }
};

set<string, LessStr> s = {"apple", "banana"};
s.find("apple");          // finds without constructing a temporary string  // C++14
```

(For `unordered_*`, heterogeneous lookup arrived in C++20.)

---

## Small but useful: `cbegin`, `cend` non-member forms

C++11 had `begin(c)` and `end(c)`; C++14 added `cbegin(c)`, `cend(c)`, `rbegin(c)`, `rend(c)`, `crbegin(c)`, `crend(c)`.

```cpp
for (auto it = cbegin(v); it != cend(v); ++it) cout << *it;
```

---

## Tuple addressing by type (C++14 expanded)

```cpp
tuple<int, string, double> t{1, "hi", 3.14};
get<string>(t);   // works because string appears exactly once
```

## See also
- [19 C++17 Features](19_cpp17_features.md)
- [14 Lambdas](14_function_objects_lambdas.md)
