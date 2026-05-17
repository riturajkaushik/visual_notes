# 26 — Advanced STL Topics

> Allocators, traits, SFINAE, `if constexpr`, move iterators.

## Setup
```cpp
#include <bits/stdc++.h>
using namespace std;
```

---

## Allocators

An allocator is the policy a container uses to acquire/release raw memory.

```cpp
vector<int> v;                            // uses std::allocator<int>
vector<int, std::allocator<int>> v2;      // explicit, same thing
```

### Custom allocator skeleton
```cpp
template <typename T>
struct LoggingAlloc {
    using value_type = T;
    LoggingAlloc() = default;
    template <typename U> LoggingAlloc(const LoggingAlloc<U>&) {}

    T* allocate(size_t n) {
        cout << "alloc " << n*sizeof(T) << " bytes\n";
        return static_cast<T*>(::operator new(n * sizeof(T)));
    }
    void deallocate(T* p, size_t n) noexcept {
        cout << "free  " << n*sizeof(T) << " bytes\n";
        ::operator delete(p);
    }
};
template <typename T, typename U>
bool operator==(const LoggingAlloc<T>&, const LoggingAlloc<U>&) { return true; }
template <typename T, typename U>
bool operator!=(const LoggingAlloc<T>&, const LoggingAlloc<U>&) { return false; }

vector<int, LoggingAlloc<int>> v;
v.reserve(10);
```

### When custom allocators matter
- Memory pools (arena allocation in games/engines).
- NUMA-aware placement.
- Custom alignment.
- Tracking / sandboxing.
- Containers in shared memory.

Most code never needs one — the default allocator is fine.

---

## Type traits — `<type_traits>`

Compile-time queries about types.

```cpp
is_integral<int>::value;        // true
is_integral_v<int>;             // true   // C++17 helper

is_same_v<int, int>;            // true
is_pointer_v<int*>;             // true
is_floating_point_v<double>;    // true

// Transforms
using R = remove_reference_t<int&>;            // int
using C = remove_cv_t<const int>;              // int
using D = decay_t<int(&)[3]>;                  // int*
using P = remove_pointer_t<int*>;              // int
using R2 = add_const_t<int>;                   // const int
```

### `conditional`
```cpp
template <bool B>
using IntOrLong = conditional_t<B, int, long>;

IntOrLong<true>  a;     // int
IntOrLong<false> b;     // long
```

### `enable_if`
```cpp
template <typename T>
enable_if_t<is_integral_v<T>, T>
abs_int(T x) { return x < 0 ? -x : x; }
```

---

## Iterator traits

```cpp
using It = vector<int>::iterator;
using cat   = iterator_traits<It>::iterator_category;   // random_access_iterator_tag
using val   = iterator_traits<It>::value_type;          // int
using dist  = iterator_traits<It>::difference_type;     // ptrdiff_t
```

Used internally by algorithms to pick optimized paths (e.g., `advance` is O(1) for RA, O(n) otherwise).

---

## `numeric_limits`

```cpp
numeric_limits<int>::max();             // INT_MAX
numeric_limits<int>::min();             // INT_MIN
numeric_limits<double>::infinity();
numeric_limits<double>::epsilon();
numeric_limits<long long>::max();
```

---

## `char_traits`

Customizes `basic_string` behavior — used to build, e.g., case-insensitive strings.
```cpp
struct CiTraits : char_traits<char> {
    static bool eq(char a, char b)     { return tolower(a) == tolower(b); }
    static bool lt(char a, char b)     { return tolower(a) <  tolower(b); }
    static int  compare(const char* a, const char* b, size_t n) {
        for (size_t i = 0; i < n; ++i)
            if (tolower(a[i]) != tolower(b[i]))
                return tolower(a[i]) < tolower(b[i]) ? -1 : 1;
        return 0;
    }
};
using ci_string = basic_string<char, CiTraits>;
```

---

## SFINAE

"Substitution Failure Is Not An Error." When template substitution produces an invalid type, the compiler silently drops that overload rather than erroring.

### `enable_if` SFINAE
```cpp
// Overload only for integral types
template <typename T,
          enable_if_t<is_integral_v<T>, int> = 0>
void process(T x) { cout << "integral " << x; }

template <typename T,
          enable_if_t<is_floating_point_v<T>, int> = 0>
void process(T x) { cout << "float " << x; }

process(5);       // "integral 5"
process(3.14);    // "float 3.14"
```

### `void_t` trick — detect a member
```cpp
template <typename, typename = void>
struct has_size : false_type {};

template <typename T>
struct has_size<T, void_t<decltype(declval<T>().size())>> : true_type {};

static_assert( has_size<vector<int>>::value);
static_assert(!has_size<int>::value);
```

---

## `if constexpr` (C++17) — usually better than SFINAE

```cpp
template <typename T>
void process(T x) {
    if constexpr (is_integral_v<T>)         cout << "integral " << x;
    else if constexpr (is_floating_point_v<T>) cout << "float " << x;
    else                                      cout << "other";
}
```

Replaces many SFINAE overloads with one readable function template.

---

## Move iterators

`make_move_iterator(it)` wraps an iterator so that `*it` returns an rvalue reference — algorithms then **move** elements rather than copy.

```cpp
vector<string> src = {"hello", "world", "!"};
vector<string> dst;
dst.reserve(src.size());
copy(make_move_iterator(src.begin()),
     make_move_iterator(src.end()),
     back_inserter(dst));
// src elements are now in moved-from state; dst owns them.
```

### Difference: `copy` vs `move` algorithms
```cpp
move(src.begin(), src.end(), back_inserter(dst));   // same effect
```
`std::move` (the algorithm in `<algorithm>`) is sugar for `copy(make_move_iterator(...), ...)`.

---

## Common pitfalls
- Mixing allocators of the same container type — they must be equal for assignment/swap to be cheap.
- Holding `iterator_traits<It>::reference` past the lifetime of the iterator.
- Using `enable_if_t` in the *return type* hides the failure better than in template params for overload resolution.

## See also
- [27 Debugging](27_debugging.md)
- [19 C++17 Features](19_cpp17_features.md) — `if constexpr`, trait `_v` helpers
