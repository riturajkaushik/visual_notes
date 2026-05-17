# 22 — Common STL Idioms

> Patterns that come up over and over.

## Setup
```cpp
#include <bits/stdc++.h>
using namespace std;
```

---

## Erase-remove idiom

`remove` and `remove_if` don't change container size — they only move the kept elements to the front and return a "new end" iterator.

```cpp
v.erase(remove(v.begin(), v.end(), x), v.end());                 // all x
v.erase(remove_if(v.begin(), v.end(),
                  [](int n){ return n < 0; }), v.end());          // matching predicate
```

For `string`:
```cpp
string s = "h e l l o";
s.erase(remove(s.begin(), s.end(), ' '), s.end());   // "hello"
```

For `list` / `forward_list`, prefer `list.remove(x)` / `list.remove_if(pred)`.

---

## Sort and unique (full deduplication)

`unique` only collapses *consecutive* duplicates, so sort first.
```cpp
sort(v.begin(), v.end());
v.erase(unique(v.begin(), v.end()), v.end());
```

For an `unordered_set` round-trip (less code, but reorders):
```cpp
unordered_set<int> s(v.begin(), v.end());
v.assign(s.begin(), s.end());
```

---

## Frequency counting

```cpp
unordered_map<int,int> freq;
for (int x : v) freq[x]++;

// top-3 most frequent
vector<pair<int,int>> items(freq.begin(), freq.end());
partial_sort(items.begin(), items.begin() + 3, items.end(),
             [](auto& a, auto& b){ return a.second > b.second; });
```

For chars:
```cpp
array<int, 256> cnt{};
for (unsigned char c : s) cnt[c]++;
```

---

## Prefix sum

For sum-of-range queries in O(1) after O(n) preprocessing:
```cpp
int n = a.size();
vector<long long> pref(n + 1, 0);
for (int i = 0; i < n; ++i) pref[i + 1] = pref[i] + a[i];

// sum of a[l..r] inclusive
auto range_sum = [&](int l, int r){ return pref[r + 1] - pref[l]; };
```

Using `partial_sum`:
```cpp
vector<long long> ps(a.size());
partial_sum(a.begin(), a.end(), ps.begin());
// ps[i] = a[0] + ... + a[i]
```

---

## Coordinate compression

Map original values into `0..k-1` based on sorted order.
```cpp
vector<int> vals = a;                                   // copy
sort(vals.begin(), vals.end());
vals.erase(unique(vals.begin(), vals.end()), vals.end());

auto compress = [&](int x){
    return int(lower_bound(vals.begin(), vals.end(), x) - vals.begin());
};

vector<int> compressed(a.size());
for (size_t i = 0; i < a.size(); ++i)
    compressed[i] = compress(a[i]);
```

---

## Two-pointer (sorted array)

### Pair sum equals K
```cpp
sort(a.begin(), a.end());
int l = 0, r = a.size() - 1;
while (l < r) {
    int s = a[l] + a[r];
    if (s == K) { /* found */ break; }
    if (s < K) ++l; else --r;
}
```

### Sliding window — distinct elements ≤ K
```cpp
unordered_map<int,int> cnt;
int l = 0, best = 0;
for (int r = 0; r < (int)a.size(); ++r) {
    cnt[a[r]]++;
    while ((int)cnt.size() > K) {
        if (--cnt[a[l]] == 0) cnt.erase(a[l]);
        ++l;
    }
    best = max(best, r - l + 1);
}
```

### Sliding window maximum (deque)
```cpp
vector<int> sliding_max(const vector<int>& a, int k) {
    vector<int> out;
    deque<int> dq;   // stores indices, values decreasing front-to-back
    for (int i = 0; i < (int)a.size(); ++i) {
        while (!dq.empty() && dq.front() <= i - k) dq.pop_front();
        while (!dq.empty() && a[dq.back()] <= a[i]) dq.pop_back();
        dq.push_back(i);
        if (i >= k - 1) out.push_back(a[dq.front()]);
    }
    return out;
}
```

---

## `lower_bound` on containers vs std::lower_bound

```cpp
// vector: either works
auto it = lower_bound(v.begin(), v.end(), x);

// set/map: MUST use the member function (the generic version is O(n))
auto it = s.lower_bound(x);
auto it = m.lower_bound(x);
```

---

## Read entire input

```cpp
// All ints from stdin
vector<int> v((istream_iterator<int>(cin)), istream_iterator<int>());

// Whole file
ifstream f("input.txt");
stringstream ss;
ss << f.rdbuf();
string contents = ss.str();

// Line by line
string line;
while (getline(cin, line)) { /* ... */ }
```

---

## Indices sorted by value

```cpp
vector<int> idx(a.size());
iota(idx.begin(), idx.end(), 0);
sort(idx.begin(), idx.end(), [&](int i, int j){ return a[i] < a[j]; });
```

---

## Swap idiom for "clear capacity"

```cpp
vector<int> v(1'000'000);
// ... done with v ...
vector<int>().swap(v);   // releases capacity (shrink_to_fit is only a hint)
```

---

## Min/max with assign

```cpp
best = min(best, candidate);
best = max(best, candidate);
// or, with structured logic
if (candidate < best) best = candidate;
```

---

## Check exists in container

```cpp
if (s.count(x)) {}                            // set / map
if (us.find(x) != us.end()) {}                // unordered_*
if (binary_search(v.begin(), v.end(), x)) {}  // sorted vector
```

## See also
- [10 Sorting & Search](10_algorithms_sorting_searching.md)
- [13 Numeric](13_numeric_algorithms.md)
- [23 Competitive Programming](23_competitive_programming.md)
