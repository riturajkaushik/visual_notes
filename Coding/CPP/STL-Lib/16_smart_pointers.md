# 16 — Smart Pointers

> `unique_ptr`, `shared_ptr`, `weak_ptr` from `<memory>`.

## Setup
```cpp
#include <bits/stdc++.h>
using namespace std;
```

---

## `std::unique_ptr` — exclusive ownership

```cpp
unique_ptr<int> p = make_unique<int>(42);     // C++14
cout << *p;          // 42
cout << p.get();     // raw pointer

p.reset(new int(99));    // free old, own new
p.reset();               // free, p == nullptr
```

### Move-only — cannot copy
```cpp
unique_ptr<int> a = make_unique<int>(1);
// unique_ptr<int> b = a;             // compile error
unique_ptr<int> b = std::move(a);     // OK; a becomes nullptr
```

### Use as class member or factory return
```cpp
struct Engine { /*...*/ };
unique_ptr<Engine> make_engine() { return make_unique<Engine>(); }

class Car {
    unique_ptr<Engine> engine_;
public:
    Car() : engine_(make_unique<Engine>()) {}
};
```

### Array form
```cpp
unique_ptr<int[]> arr = make_unique<int[]>(10);   // zero-initialized
arr[0] = 1;
arr[9] = 99;
```

### Custom deleter
```cpp
auto file_deleter = [](FILE* f){ if (f) fclose(f); };
unique_ptr<FILE, decltype(file_deleter)> fp(fopen("x.txt", "r"), file_deleter);
```

**Cost:** zero overhead vs raw pointer + manual delete. Default choice for owning a heap-allocated resource.

---

## `std::shared_ptr` — shared ownership, reference-counted

```cpp
shared_ptr<int> a = make_shared<int>(42);
shared_ptr<int> b = a;     // both share the same int; refcount = 2
a.use_count();             // 2
b.reset();                 // refcount = 1
a.reset();                 // refcount = 0, int is freed
```

### `make_shared` vs `new`
```cpp
shared_ptr<Foo> x = make_shared<Foo>(args...);   // 1 allocation, faster
shared_ptr<Foo> y(new Foo(args...));              // 2 allocations (control block + object)
```
Prefer `make_shared` unless you need a custom deleter (which `make_shared` can't take).

### When to use
- Multiple owners that may all release the object at unpredictable times.
- Asynchronous tasks that may outlive the caller.
- **Don't use as default** — `unique_ptr` is cheaper and clearer.

### Thread safety
- The reference count itself is thread-safe (atomic).
- The **pointed-to object** is NOT — protect access with a mutex if you mutate from multiple threads.

### Cyclic reference problem
```cpp
struct Node {
    shared_ptr<Node> next;
};
auto a = make_shared<Node>();
auto b = make_shared<Node>();
a->next = b;
b->next = a;        // CYCLE: refcounts never reach 0 — memory leak
```
Solution: make one direction a `weak_ptr`.

---

## `std::weak_ptr` — non-owning observer

```cpp
auto sp = make_shared<int>(42);
weak_ptr<int> wp = sp;          // doesn't bump refcount

if (auto locked = wp.lock()) {  // shared_ptr or null
    cout << *locked;
} else {
    cout << "expired";
}

wp.expired();                   // true if the object is gone
```

### Breaking cycles
```cpp
struct Node {
    shared_ptr<Node> next;
    weak_ptr<Node>   prev;   // break the cycle
};
```

### Common use: cache observer
```cpp
unordered_map<string, weak_ptr<Image>> cache;

shared_ptr<Image> get(const string& key) {
    if (auto sp = cache[key].lock()) return sp;     // still alive
    auto sp = load_image(key);
    cache[key] = sp;
    return sp;
}
```

---

## Cheat sheet

| Need | Use |
|---|---|
| Unique heap object | `unique_ptr` + `make_unique` (C++14) |
| Multiple owners | `shared_ptr` + `make_shared` |
| Observer that doesn't extend lifetime | `weak_ptr` |
| Pass non-owning pointer to a function | raw pointer or reference, not a smart pointer |
| Pass ownership into a function | by-value `unique_ptr` (sink) |

### Rule of thumb for function params
```cpp
void process(unique_ptr<Foo> p);    // takes ownership
void process(shared_ptr<Foo> p);    // takes a share
void process(Foo* p);               // borrows, may be null
void process(Foo& f);               // borrows, never null
```

## See also
- [01 Foundations](01_foundations.md) — RAII basics
- [18 C++14 Features](18_cpp14_features.md) — `make_unique`
