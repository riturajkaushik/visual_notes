# 24 — Complexity Cheat Sheet

> Big-O for the operations you actually use.

---

## Containers

### Random access / lookup

| Container | `[i]` / `at` | `find(k)` | `insert` | `erase` |
|---|---|---|---|---|
| `vector` | O(1) | O(n) | O(n) middle, amortized O(1) at end | O(n) middle, O(1) at end |
| `array` | O(1) | O(n) | — | — |
| `deque` | O(1) | O(n) | O(n) middle, O(1) at ends | O(n) middle, O(1) at ends |
| `list` / `forward_list` | — | O(n) | O(1) with iterator | O(1) with iterator |
| `set` / `map` (sorted) | — | O(log n) | O(log n) | O(log n) |
| `multiset` / `multimap` | — | O(log n) | O(log n) | O(log n) |
| `unordered_set` / `unordered_map` | — | avg O(1), worst O(n) | avg O(1) | avg O(1) |
| `unordered_multiset` / `unordered_multimap` | — | avg O(1) | avg O(1) | avg O(1) |
| `priority_queue` | `top()` O(1) | — | push O(log n) | pop O(log n) |
| `stack`, `queue` | top/front/back O(1) | — | push O(1) | pop O(1) |
| `bitset<N>` | `[i]` O(1); ops over all bits O(N/64) | — | — | — |

### Iteration
All sequence and ordered containers: O(n) for full traversal. `unordered_*` also O(n) but with bucket overhead.

---

## Memory layout notes

| Container | Layout | Cache-friendly? |
|---|---|---|
| `vector`, `array` | contiguous | yes |
| `deque` | array of fixed blocks | partially |
| `list` | doubly linked | no |
| `forward_list` | singly linked | no |
| `set`/`map` | red-black tree (typically) | no |
| `unordered_*` | hash table with chained buckets | partially |

In practice, **`vector` beats `list` for most middle-insertion workloads up to surprisingly large n** because of cache locality. Reach for `list` only when you have a measured reason.

---

## Algorithms (`<algorithm>` / `<numeric>`)

### Non-modifying
| Algorithm | Complexity |
|---|---|
| `find`, `find_if`, `count`, `count_if` | O(n) |
| `all_of`, `any_of`, `none_of`, `for_each` | O(n) |
| `equal`, `mismatch` | O(n) |
| `search`, `find_end`, `find_first_of` | O(n·m) worst |
| `adjacent_find` | O(n) |

### Modifying
| Algorithm | Complexity |
|---|---|
| `copy`, `move`, `fill`, `generate`, `transform`, `replace` | O(n) |
| `remove`, `remove_if`, `unique` | O(n) |
| `reverse`, `rotate`, `swap_ranges` | O(n) |
| `shuffle`, `sample` (C++17) | O(n) |

### Sort / search (sorted ranges)
| Algorithm | Complexity |
|---|---|
| `sort` | O(n log n) |
| `stable_sort` | O(n log n) — may use extra mem |
| `partial_sort` | O(n log k) |
| `nth_element` | average O(n) |
| `is_sorted` | O(n) |
| `binary_search`, `lower_bound`, `upper_bound`, `equal_range` | O(log n) on RA iters; O(n) advancement on others |
| `merge` | O(n + m) |
| `inplace_merge` | O(n) with extra mem, O(n log n) without |

### Set operations (sorted ranges)
| Algorithm | Complexity |
|---|---|
| `set_union`, `set_intersection`, `set_difference`, `set_symmetric_difference`, `includes` | O(n + m) |

### Heap
| Algorithm | Complexity |
|---|---|
| `make_heap` | O(n) |
| `push_heap`, `pop_heap` | O(log n) |
| `sort_heap` | O(n log n) |
| `is_heap` | O(n) |

### Min/max
| Algorithm | Complexity |
|---|---|
| `min`, `max`, `clamp` | O(1) |
| `min_element`, `max_element`, `minmax_element` | O(n) |

### Permutation / partition
| Algorithm | Complexity |
|---|---|
| `next_permutation`, `prev_permutation` | O(n) per call |
| `is_permutation` | O(n²) worst |
| `partition` | O(n) |
| `stable_partition` | O(n) with mem, O(n log n) without |
| `partition_point` | O(log n) |

### Numeric
| Algorithm | Complexity |
|---|---|
| `accumulate`, `reduce` | O(n) |
| `inner_product`, `transform_reduce` | O(n) |
| `partial_sum`, `inclusive_scan`, `exclusive_scan` | O(n) |
| `adjacent_difference` | O(n) |
| `iota` | O(n) |
| `gcd`, `lcm` | O(log min(a,b)) |

---

## Iterator invalidation cheat sheet

| Container | Insert invalidates | Erase invalidates |
|---|---|---|
| `vector` | all if realloc; else iters ≥ pos | iters ≥ erased pos |
| `deque` | all | all (except ends for end ops) |
| `list`, `forward_list` | nothing | only the erased iter |
| `set`, `map`, `multi*` | nothing | only the erased iter |
| `unordered_*` | all if rehash | only the erased iter; refs/ptrs survive |

---

## Rough budget per second (modern CPU)
- Simple integer ops: 10^8–10^9 per second
- `unordered_map` lookup: 10^7–10^8 per second
- `map`/`set` op: 10^7 per second
- Heap push/pop: 10^7 per second

Use these to back-of-envelope whether O(n log n) at n=10^6 will fit in your time limit.

## See also
- [02–06](02_sequence_containers.md) container files
- [08–13](08_algorithms_non_modifying.md) algorithm files
