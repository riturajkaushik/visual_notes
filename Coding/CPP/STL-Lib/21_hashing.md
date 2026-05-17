# 21 — Hashing & Custom Hash Functions

> How `unordered_*` works, and how to plug in hash & equality for custom keys.

## Setup
```cpp
#include <bits/stdc++.h>
using namespace std;
```

---

## Hashing basics

An `unordered_map<K, V>` uses:
- a **hash function** `H(K) -> size_t` to bucket the key
- an **equality function** `Eq(K, K) -> bool` to disambiguate keys within a bucket

Internals:
- `bucket_count()` — number of buckets
- `load_factor() = size() / bucket_count()`
- `max_load_factor()` — when exceeded, rehash doubles bucket count
- `rehash(n)` / `reserve(n)` — pre-size

```cpp
unordered_map<int,int> m;
m.reserve(1'000'000);             // avoid repeated rehashes
m.max_load_factor(0.7);           // tune density / lookup speed
```

**Collision handling:** separate chaining (linked list per bucket). Worst-case is O(n) when all keys collide.

---

## Built-in `std::hash` specializations

Works out of the box for: integers, floats, pointers, `string`, `string_view`, `bitset`, `unique_ptr`, `shared_ptr`, `thread::id`, `error_code`.

**Doesn't work** out of the box for: `pair`, `tuple`, your own structs, `vector`. You must provide a hash.

---

## Custom hash for `pair<int,int>`

```cpp
struct PairHash {
    size_t operator()(const pair<int,int>& p) const noexcept {
        // Pack two 32-bit ints into a 64-bit number, then hash
        return hash<long long>()(
            (long long)p.first * 1'000'003LL + (long long)p.second);
    }
};

unordered_map<pair<int,int>, int, PairHash> mp;
mp[{1, 2}] = 100;
```

**Don't do this:**
```cpp
return hash<int>()(p.first) ^ hash<int>()(p.second);   // BAD
// XOR of equal values gives 0; many symmetric pairs collide
```

Better mix:
```cpp
struct PairHash {
    size_t operator()(const pair<int,int>& p) const noexcept {
        auto h1 = hash<int>()(p.first);
        auto h2 = hash<int>()(p.second);
        return h1 * 31 + h2;                  // simple but better than XOR
    }
};
```

---

## Custom hash for a struct

```cpp
struct Point { int x, y; };

struct PointHash {
    size_t operator()(const Point& p) const noexcept {
        size_t h = hash<int>()(p.x);
        h ^= hash<int>()(p.y) + 0x9e3779b9 + (h << 6) + (h >> 2);  // boost-style mix
        return h;
    }
};

struct PointEq {
    bool operator()(const Point& a, const Point& b) const noexcept {
        return a.x == b.x && a.y == b.y;
    }
};

unordered_set<Point, PointHash, PointEq> seen;
unordered_map<Point, int, PointHash, PointEq> grid;
```

Alternative: define `operator==` on `Point` and you only need the hash:
```cpp
struct Point {
    int x, y;
    bool operator==(const Point& o) const { return x == o.x && y == o.y; }
};
unordered_set<Point, PointHash> seen;
```

---

## Hash for a `vector` or other container

```cpp
struct VecHash {
    size_t operator()(const vector<int>& v) const noexcept {
        size_t h = v.size();
        for (int x : v)
            h ^= hash<int>()(x) + 0x9e3779b9 + (h << 6) + (h >> 2);
        return h;
    }
};
unordered_set<vector<int>, VecHash> seen;
```

---

## Specialize `std::hash` (alternative style)

You can plug into `std::hash` instead of passing a separate type.

```cpp
struct Point { int x, y; bool operator==(const Point& o) const = default; };

namespace std {
    template <>
    struct hash<Point> {
        size_t operator()(const Point& p) const noexcept {
            return hash<long long>()(((long long)p.x << 32) ^ (unsigned)p.y);
        }
    };
}

unordered_set<Point> s;       // no extra template args needed
```

(C++20 brought defaulted `==`; in C++14/17 write it manually.)

---

## Anti-hash attacks in competitive programming

Codeforces (and others) include test cases that force `unordered_map<int,int>` worst-case. Use a randomized hash:

```cpp
struct CustomHash {
    size_t operator()(uint64_t x) const noexcept {
        static const uint64_t FIXED = chrono::steady_clock::now()
                                          .time_since_epoch().count();
        x ^= FIXED;
        x = (x ^ (x >> 30)) * 0xbf58476d1ce4e5b9ULL;
        x = (x ^ (x >> 27)) * 0x94d049bb133111ebULL;
        x ^= x >> 31;
        return x;
    }
};
unordered_map<long long, int, CustomHash> safe_map;
```

---

## Tuning checklist

| Symptom | Action |
|---|---|
| Many rehashes during bulk insert | `reserve(n)` before inserts |
| Lookups slow | Print `bucket_count`, `load_factor`; consider better hash |
| Worst-case in CP | Use the randomized hash above |
| Map of complex keys (`pair`, struct) | Write hash + equality, or specialize `std::hash` |
| Need ordered iteration | Use `map`/`set` instead |

## See also
- [04 Unordered Containers](04_unordered_containers.md)
- [20 Comparators](20_comparators.md) — for the ordered side
