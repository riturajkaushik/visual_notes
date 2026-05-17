# C++ STL Tutorial — C++14 / C++17

A code-first reference. Each file is independent and self-contained.

## 1. Foundations
- [01 — Foundations](01_foundations.md)

## 2. Containers
- [02 — Sequence Containers](02_sequence_containers.md) — `vector`, `array`, `deque`, `list`, `forward_list`
- [03 — Associative Containers](03_associative_containers.md) — `set`, `multiset`, `map`, `multimap`
- [04 — Unordered Containers](04_unordered_containers.md) — `unordered_*`
- [05 — Container Adapters](05_container_adapters.md) — `stack`, `queue`, `priority_queue`
- [06 — Strings & string_view](06_strings.md) — `string`, `string_view` (C++17)

## 3. Iterators
- [07 — Iterators](07_iterators.md)

## 4. Algorithms
- [08 — Non-Modifying Algorithms](08_algorithms_non_modifying.md)
- [09 — Modifying Algorithms](09_algorithms_modifying.md)
- [10 — Sorting & Binary Search](10_algorithms_sorting_searching.md)
- [11 — Set, Heap, Min/Max](11_algorithms_set_heap_minmax.md)
- [12 — Permutation & Partition](12_algorithms_permutation_partition.md)

## 5. Numeric
- [13 — Numeric Algorithms](13_numeric_algorithms.md)

## 6. Function Objects & Callables
- [14 — Lambdas & Function Objects](14_function_objects_lambdas.md)

## 7. Pair, Tuple, Structured Bindings
- [15 — Pair, Tuple, Bindings](15_pair_tuple_bindings.md)

## 8. Memory
- [16 — Smart Pointers](16_smart_pointers.md)

## 9. Utility Components
- [17 — Utility Components](17_utility_components.md) — `optional`, `variant`, `any`, `filesystem`, `chrono`, `<random>`

## 10–11. Language Feature Highlights
- [18 — C++14 Features](18_cpp14_features.md)
- [19 — C++17 Features](19_cpp17_features.md)

## 12–13. Custom Ordering & Hashing
- [20 — Comparators](20_comparators.md)
- [21 — Hashing & Custom Hash Functions](21_hashing.md)

## 14. Idioms
- [22 — Common STL Idioms](22_idioms.md)

## 15–16. Practical Use
- [23 — Competitive Programming](23_competitive_programming.md)
- [24 — Complexity Cheat Sheet](24_complexity.md)

## 17. Parallelism
- [25 — Parallel Algorithms (C++17)](25_parallel_algorithms.md)

## 18–19. Advanced & Debugging
- [26 — Advanced Topics](26_advanced_topics.md) — allocators, traits, SFINAE, move iterators
- [27 — Debugging STL Code](27_debugging.md)

## 20–23. Wrap-up
- [28 — Practice, Snippets & Checklist](28_practice_and_checklist.md)

---

## Suggested Study Order

1. **Core basics** → [01 Foundations](01_foundations.md), [02 Sequence Containers](02_sequence_containers.md), [06 Strings](06_strings.md), [07 Iterators](07_iterators.md), [10 Sorting & Search](10_algorithms_sorting_searching.md), [15 Pair/Tuple](15_pair_tuple_bindings.md)
2. **CP essentials** → [03 Associative](03_associative_containers.md), [04 Unordered](04_unordered_containers.md), [05 Adapters](05_container_adapters.md), [13 Numeric](13_numeric_algorithms.md), [22 Idioms](22_idioms.md)
3. **Intermediate** → [14 Lambdas](14_function_objects_lambdas.md), [20 Comparators](20_comparators.md), [08 Non-Modifying](08_algorithms_non_modifying.md), [09 Modifying](09_algorithms_modifying.md), [11 Set/Heap/MinMax](11_algorithms_set_heap_minmax.md), [12 Perm/Partition](12_algorithms_permutation_partition.md)
4. **Modern C++** → [18 C++14](18_cpp14_features.md), [19 C++17](19_cpp17_features.md), [16 Smart Pointers](16_smart_pointers.md), [17 Utilities](17_utility_components.md)
5. **Advanced** → [21 Hashing](21_hashing.md), [25 Parallel](25_parallel_algorithms.md), [26 Advanced](26_advanced_topics.md), [27 Debugging](27_debugging.md), [24 Complexity](24_complexity.md), [23 CP Patterns](23_competitive_programming.md), [28 Practice](28_practice_and_checklist.md)

---

## Conventions used in these files

- Snippets assume `#include <bits/stdc++.h>` and `using namespace std;` (shown explicitly in each file's Setup block).
- `// C++14` and `// C++17` tags mark version-specific features.
- `// => ...` comments show expected output.
- **Complexity:** lines call out algorithmic cost when it matters.
- **Gotcha:** lines flag iterator invalidation, UB, or footguns.
