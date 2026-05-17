# 17 — Utility Components

> `optional`, `variant`, `any` (C++17); `filesystem` (C++17); `chrono`; `<random>`. `string_view` lives in [06](06_strings.md).

## Setup
```cpp
#include <bits/stdc++.h>
using namespace std;
```

---

## `std::optional` (C++17)

A value-or-nothing wrapper. No allocation.

```cpp
optional<int> find_first_negative(const vector<int>& v) {
    for (int x : v) if (x < 0) return x;
    return nullopt;
}

auto r = find_first_negative({1, 2, -3, 4});
if (r) cout << *r;                       // -3
if (r.has_value()) cout << r.value();    // same

cout << r.value_or(0);    // -3 (or 0 if no value)
```

### Build & reset
```cpp
optional<string> s;          // empty
s = "hi";
s.emplace("world");          // construct in-place
s.reset();                   // empty again
```

### Use cases — replaces sentinel returns
```cpp
// Before: return -1 means "not found"
int find_idx_bad(const vector<int>& v, int x);

// After: meaning is in the type
optional<size_t> find_idx(const vector<int>& v, int x);
```

**Gotcha:** `*opt` and `opt->...` are UB if empty. Check first, or use `value()` for the throwing version.

---

## `std::variant` (C++17)

A type-safe union: holds exactly one of N declared types.

```cpp
variant<int, double, string> v = 42;
cout << get<int>(v);              // 42

v = 3.14;
cout << get<double>(v);

v = string("hello");
cout << get<string>(v);

holds_alternative<int>(v);        // false (currently holds string)
v.index();                        // 2  (0=int, 1=double, 2=string)
```

### Safe access
```cpp
if (auto p = get_if<int>(&v)) {
    cout << *p;
} else {
    cout << "not an int";
}
```
`get<T>` throws `bad_variant_access` if wrong type; `get_if` returns pointer or null.

### `visit` — dispatch on the active type
```cpp
variant<int, double, string> v = 3.14;
visit([](auto&& arg){
    using T = decay_t<decltype(arg)>;
    if constexpr (is_same_v<T, int>)         cout << "int " << arg;
    else if constexpr (is_same_v<T, double>) cout << "dbl " << arg;
    else                                      cout << "str " << arg;
}, v);
```

### Use cases
- Sum types ("either X or Y or Z")
- AST nodes
- State machines
- Replaces inheritance for closed sets of types

---

## `std::any` (C++17)

Holds any copyable value of any type, accessed by `any_cast<T>`. Erases the type at compile time.

```cpp
any a = 42;
cout << any_cast<int>(a);      // 42

a = string("hi");
cout << any_cast<string>(a);

try { any_cast<int>(a); }
catch (const bad_any_cast& e) { cout << "wrong type"; }

a.reset();                    // empty
a.has_value();                // false
```

**When to use:** rarely. Prefer `variant` (closed set) or template parameters (open set). `any` is mostly for plugin / message-bus boundaries where types truly aren't known statically.

---

## `std::filesystem` (C++17)

```cpp
namespace fs = std::filesystem;

fs::path p = "/tmp/foo/bar.txt";
p.parent_path();           // /tmp/foo
p.filename();              // bar.txt
p.stem();                  // bar
p.extension();             // .txt
p /= "child";              // /tmp/foo/bar.txt/child   (path append)

fs::exists(p);
fs::is_regular_file(p);
fs::is_directory(p);
fs::file_size(p);

fs::create_directory("/tmp/new");
fs::create_directories("/tmp/a/b/c");
fs::remove("/tmp/foo.txt");
fs::remove_all("/tmp/foo");      // recursive

// Iterate a directory
for (const auto& entry : fs::directory_iterator("/tmp")) {
    cout << entry.path() << '\n';
}
for (const auto& entry : fs::recursive_directory_iterator("/tmp")) {
    cout << entry.path() << '\n';
}
```

**Build note:** older g++ requires `-lstdc++fs`; clang may need `-lc++fs`. Modern compilers don't.

---

## `std::chrono`

Three pieces: **durations**, **time points**, **clocks**.

### Durations
```cpp
using namespace std::chrono;

auto d1 = milliseconds(500);
auto d2 = seconds(2);
auto sum = d1 + d2;                 // 2500ms
auto ms_value = sum.count();        // 2500

// C++14 chrono literals
auto t = 100ms;
auto u = 2s;
this_thread::sleep_for(100ms);
```

### Clocks & time points
```cpp
auto now = system_clock::now();
auto tt  = system_clock::to_time_t(now);
cout << ctime(&tt);

steady_clock::time_point start = steady_clock::now();
// ... work ...
steady_clock::time_point end = steady_clock::now();
auto dur = end - start;
cout << duration_cast<milliseconds>(dur).count() << " ms";
```

### Which clock?
- `steady_clock` — monotonic, **always use for elapsed time**.
- `system_clock` — wall clock; can jump (NTP, user change).
- `high_resolution_clock` — usually an alias for one of the above.

### Timing helper
```cpp
template <typename F, typename... A>
auto time_ms(F&& f, A&&... args) {
    auto start = steady_clock::now();
    forward<F>(f)(forward<A>(args)...);
    return duration_cast<milliseconds>(steady_clock::now() - start).count();
}
```

---

## `<random>`

```cpp
random_device rd;                          // entropy source
mt19937 rng(rd());                         // 32-bit Mersenne Twister
mt19937_64 rng64(rd());

uniform_int_distribution<int> dist(1, 6);  // inclusive
int dice = dist(rng);

uniform_real_distribution<double> ud(0.0, 1.0);
double r = ud(rng);

// Shuffle with a real engine (replaces deprecated random_shuffle)
vector<int> v = {1,2,3,4,5};
shuffle(v.begin(), v.end(), rng);
```

### Reproducibility
```cpp
mt19937 rng(12345);   // fixed seed → same sequence every run
```

### Other distributions
- `normal_distribution<double>(mean, stddev)`
- `bernoulli_distribution(p)`
- `binomial_distribution<int>(n, p)`
- `discrete_distribution<int>({w1, w2, w3})`

**CP tip:** keep one global `mt19937` per program; never construct distributions inside a hot loop unnecessarily.

## See also
- [06 Strings](06_strings.md) — `string_view`
- [19 C++17 Features](19_cpp17_features.md)
