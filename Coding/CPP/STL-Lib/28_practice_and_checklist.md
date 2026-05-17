# 28 — Practice, Snippets & Final Checklist

> Suggested study order, must-know snippets, problem categories, and a self-assessment list.

---

## Suggested study order

### Phase 1 — Core basics
1. `vector` — [02](02_sequence_containers.md)
2. `string` — [06](06_strings.md)
3. Iterators — [07](07_iterators.md)
4. Range-based loops — [01](01_foundations.md)
5. `sort` — [10](10_algorithms_sorting_searching.md)
6. `find` — [08](08_algorithms_non_modifying.md)
7. `reverse` — [09](09_algorithms_modifying.md)
8. `min_element`, `max_element` — [11](11_algorithms_set_heap_minmax.md)
9. `accumulate` — [13](13_numeric_algorithms.md)
10. `pair` — [15](15_pair_tuple_bindings.md)

### Phase 2 — CP essentials
1. `set` — [03](03_associative_containers.md)
2. `map` — [03](03_associative_containers.md)
3. `unordered_map` — [04](04_unordered_containers.md)
4. `unordered_set` — [04](04_unordered_containers.md)
5. `stack` — [05](05_container_adapters.md)
6. `queue` — [05](05_container_adapters.md)
7. `priority_queue` — [05](05_container_adapters.md)
8. `lower_bound` — [10](10_algorithms_sorting_searching.md)
9. `upper_bound` — [10](10_algorithms_sorting_searching.md)
10. `next_permutation` — [12](12_algorithms_permutation_partition.md)

### Phase 3 — Intermediate
1. Custom comparators — [20](20_comparators.md)
2. Lambdas — [14](14_function_objects_lambdas.md)
3. `deque` — [02](02_sequence_containers.md)
4. `list` — [02](02_sequence_containers.md)
5. `tuple` — [15](15_pair_tuple_bindings.md)
6. `bitset` — [23](23_competitive_programming.md)
7. `unique` — [09](09_algorithms_modifying.md)
8. Erase-remove idiom — [22](22_idioms.md)
9. `nth_element` — [10](10_algorithms_sorting_searching.md)
10. Heap algorithms — [11](11_algorithms_set_heap_minmax.md)

### Phase 4 — C++14 / C++17
1. Generic lambdas — [18](18_cpp14_features.md)
2. `make_unique` — [16](16_smart_pointers.md), [18](18_cpp14_features.md)
3. Structured bindings — [15](15_pair_tuple_bindings.md), [19](19_cpp17_features.md)
4. `if constexpr` — [19](19_cpp17_features.md)
5. `optional` — [17](17_utility_components.md)
6. `variant` — [17](17_utility_components.md)
7. `string_view` — [06](06_strings.md)
8. `filesystem` — [17](17_utility_components.md)
9. Parallel algorithms — [25](25_parallel_algorithms.md)
10. New numeric algorithms (`reduce`, `gcd`/`lcm`, scans) — [13](13_numeric_algorithms.md)

### Phase 5 — Advanced
1. Iterator invalidation — [07](07_iterators.md)
2. Allocators — [26](26_advanced_topics.md)
3. Type traits — [26](26_advanced_topics.md)
4. SFINAE — [26](26_advanced_topics.md)
5. Custom hash functions — [21](21_hashing.md)
6. Move iterators — [26](26_advanced_topics.md)
7. Performance tuning — [24](24_complexity.md)
8. Parallel execution safety — [25](25_parallel_algorithms.md)

---

## Must-know snippets

```cpp
// Sort ascending / descending
sort(v.begin(), v.end());
sort(v.begin(), v.end(), greater<int>());

// Remove all occurrences of x
v.erase(remove(v.begin(), v.end(), x), v.end());

// Remove duplicates
sort(v.begin(), v.end());
v.erase(unique(v.begin(), v.end()), v.end());

// Frequency map
unordered_map<int,int> freq;
for (int x : v) freq[x]++;

// Index of first element >= x
int idx = lower_bound(v.begin(), v.end(), x) - v.begin();

// Min heap
priority_queue<int, vector<int>, greater<int>> pq;

// Iterate map (C++17)
for (auto& [k, v] : mp) cout << k << ' ' << v << '\n';

// Sum with long long
long long sum = accumulate(v.begin(), v.end(), 0LL);

// Generate sequence
vector<int> v(n);
iota(v.begin(), v.end(), 1);   // 1, 2, ..., n

// Indices sorted by value
vector<int> idx(n);
iota(idx.begin(), idx.end(), 0);
sort(idx.begin(), idx.end(), [&](int i, int j){ return a[i] < a[j]; });

// Prefix sum
vector<long long> pref(n+1, 0);
for (int i = 0; i < n; ++i) pref[i+1] = pref[i] + a[i];

// Read all ints from stdin
vector<int> v((istream_iterator<int>(cin)), istream_iterator<int>());

// Coordinate compression
vector<int> vals = a;
sort(vals.begin(), vals.end());
vals.erase(unique(vals.begin(), vals.end()), vals.end());
auto cz = [&](int x){ return lower_bound(vals.begin(), vals.end(), x) - vals.begin(); };

// Top K with nth_element
nth_element(v.begin(), v.begin() + (k-1), v.end(), greater<>());

// Sliding window max (deque) — see [22 Idioms]
```

---

## Practice problem categories

### Containers
- Dynamic array operations
- Frequency counting
- Duplicate removal
- Ordered set queries
- Hash map lookup
- Priority queue scheduling

### Algorithms
- Sorting custom objects
- Binary search
- Permutations
- Heap operations
- Partitioning
- Prefix sums
- Range transformation

### Common problem types
- Two sum
- Top K frequent elements
- Merge intervals
- Kth largest element
- Longest substring without repeating characters
- Sliding window maximum
- Meeting rooms
- Dijkstra's algorithm
- Word frequency counter
- Coordinate compression
- Next permutation
- Median from data stream

Most of these have skeletons in [23 Competitive Programming](23_competitive_programming.md) and [22 Idioms](22_idioms.md).

---

## Final checklist

You should be comfortable with:

- [ ] Choosing the right STL container for the access pattern
- [ ] Knowing iterator categories per container
- [ ] Knowing iterator invalidation rules ([07](07_iterators.md))
- [ ] Using STL algorithms on iterator ranges
- [ ] Writing custom comparators (strict weak ordering)
- [ ] Using lambdas with algorithms
- [ ] Using `map`, `set`, `unordered_map`, `priority_queue`
- [ ] Applying binary search via `lower_bound` / `upper_bound`
- [ ] Knowing complexity ([24](24_complexity.md)) of key ops
- [ ] Using C++14 generic lambdas
- [ ] Using C++14 `make_unique`
- [ ] Using C++17 structured bindings
- [ ] Using C++17 `optional`, `variant`, `any`, `string_view`
- [ ] Writing custom hash functions ([21](21_hashing.md))
- [ ] Avoiding the classic bugs in [27 Debugging](27_debugging.md)
- [ ] Reading & writing clean, modern C++14/17 STL code

## See also
- [index.md](index.md)
