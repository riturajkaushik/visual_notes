Below is a structured reading index for studying the **C++ STL library and algorithms**, with focus on **C++14 and C++17**.

# C++ STL and Algorithms Reading Index

## 1. STL Foundations

### 1.1 What is STL?

Study:

* STL design philosophy
* Generic programming
* Containers, iterators, algorithms, function objects
* Separation of data structures and algorithms
* Compile-time polymorphism using templates

### 1.2 Required C++ Basics Before STL

Study:

* Templates
* Function templates
* Class templates
* Template type deduction
* `auto`
* `decltype`
* References and const references
* Move semantics
* Lambda expressions
* Operator overloading
* RAII
* Copy vs move constructors
* Initializer lists

### 1.3 Header Files Overview

Study common STL headers:

```cpp
#include <vector>
#include <array>
#include <deque>
#include <list>
#include <forward_list>
#include <set>
#include <map>
#include <unordered_set>
#include <unordered_map>
#include <queue>
#include <stack>
#include <algorithm>
#include <numeric>
#include <iterator>
#include <functional>
#include <utility>
#include <tuple>
#include <memory>
#include <string>
#include <bitset>
#include <chrono>
#include <random>
#include <optional>   // C++17
#include <variant>    // C++17
#include <any>        // C++17
```

---

# 2. STL Containers

## 2.1 Container Categories

Study:

* Sequence containers
* Associative containers
* Unordered associative containers
* Container adapters
* Fixed-size containers
* String as a container-like type

Understand:

* Time complexity
* Memory layout
* Iterator invalidation
* Use cases
* Tradeoffs

---

## 2.2 Sequence Containers

### 2.2.1 `std::vector`

Study:

* Dynamic array concept
* Declaration and initialization
* `push_back`
* `emplace_back`
* `pop_back`
* `size`
* `capacity`
* `reserve`
* `resize`
* `clear`
* `insert`
* `erase`
* `front`
* `back`
* `data`
* Random access
* Iterator invalidation
* When to use `vector`

Important concepts:

* Contiguous memory
* Amortized O(1) insertion at end
* Reallocation
* Difference between `reserve()` and `resize()`

Practice:

```cpp
vector<int> v;
v.push_back(10);
v.emplace_back(20);
sort(v.begin(), v.end());
```

---

### 2.2.2 `std::array`

Study:

* Fixed-size array wrapper
* Difference from C-style arrays
* `size`
* `front`
* `back`
* `data`
* Iterators
* Compile-time size

Use when:

* Size is known at compile time
* You want STL-compatible fixed arrays

---

### 2.2.3 `std::deque`

Study:

* Double-ended queue
* Fast insertion/removal at both ends
* `push_front`
* `push_back`
* `pop_front`
* `pop_back`
* Random access
* Memory model
* Difference from `vector`

Use when:

* Frequent insertion/removal from both ends

---

### 2.2.4 `std::list`

Study:

* Doubly linked list
* `push_front`
* `push_back`
* `insert`
* `erase`
* `splice`
* `merge`
* `sort`
* `reverse`
* No random access
* Stable iterators

Use when:

* Frequent insertion/deletion in middle
* Iterator stability matters

---

### 2.2.5 `std::forward_list`

Study:

* Singly linked list
* `push_front`
* `insert_after`
* `erase_after`
* `before_begin`
* Lower memory overhead than `list`
* No size member before C++17 implementations may vary

Use when:

* Very memory-conscious linked-list use case

---

## 2.3 Associative Containers

Associative containers are ordered and usually implemented as balanced binary search trees.

### 2.3.1 `std::set`

Study:

* Stores unique keys
* Sorted order
* Insert, erase, find
* `lower_bound`
* `upper_bound`
* `equal_range`
* Custom comparator
* O(log n) operations

Use when:

* You need unique sorted values

---

### 2.3.2 `std::multiset`

Study:

* Allows duplicate keys
* Sorted order
* Counting duplicate values
* Range queries using `equal_range`

Use when:

* You need sorted duplicates

---

### 2.3.3 `std::map`

Study:

* Key-value pairs
* Unique sorted keys
* `operator[]`
* `at`
* `insert`
* `emplace`
* `find`
* `lower_bound`
* `upper_bound`
* Custom comparator

Important:

```cpp
map<string, int> freq;
freq["apple"]++;
```

Understand:

* `operator[]` creates a default value if key does not exist
* `at()` throws if key is missing

---

### 2.3.4 `std::multimap`

Study:

* Multiple values for same key
* `equal_range`
* Sorted key-value storage

Use when:

* One key can map to multiple values

---

## 2.4 Unordered Associative Containers

Usually implemented using hash tables.

### 2.4.1 `std::unordered_set`

Study:

* Unique values
* Average O(1) operations
* Hashing
* Buckets
* Load factor
* Rehashing
* Custom hash functions

---

### 2.4.2 `std::unordered_multiset`

Study:

* Hash table with duplicate values
* Counting occurrences
* Equal range behavior

---

### 2.4.3 `std::unordered_map`

Study:

* Key-value hash table
* Average O(1) insert, erase, find
* `operator[]`
* `at`
* `find`
* `count`
* Custom hash for pairs or structs

Example:

```cpp
unordered_map<string, int> mp;
mp["cat"]++;
```

---

### 2.4.4 `std::unordered_multimap`

Study:

* Duplicate keys
* Hash-based multiple value mapping

---

## 2.5 Container Adapters

### 2.5.1 `std::stack`

Study:

* LIFO structure
* `push`
* `pop`
* `top`
* `empty`
* `size`
* Underlying container

Use cases:

* Parentheses matching
* DFS
* Expression evaluation
* Monotonic stack problems

---

### 2.5.2 `std::queue`

Study:

* FIFO structure
* `push`
* `pop`
* `front`
* `back`
* BFS problems

Use cases:

* BFS
* Scheduling
* Level order traversal

---

### 2.5.3 `std::priority_queue`

Study:

* Heap-based container adapter
* Max heap by default
* Min heap using comparator
* Custom comparators
* Pair ordering
* Struct ordering

Examples:

```cpp
priority_queue<int> maxHeap;

priority_queue<int, vector<int>, greater<int>> minHeap;
```

Use cases:

* Dijkstra’s algorithm
* Top K elements
* Median maintenance
* Scheduling
* Greedy algorithms

---

## 2.6 String

### 2.6.1 `std::string`

Study:

* Declaration
* Concatenation
* Comparison
* Indexing
* `substr`
* `find`
* `rfind`
* `erase`
* `insert`
* `replace`
* `stoi`
* `stoll`
* `to_string`
* Iteration
* Sorting characters
* Reversing strings

Practice:

```cpp
string s = "hello";
reverse(s.begin(), s.end());
```

### 2.6.2 C++17 String Additions

Study:

* `std::string_view`
* Difference between `string` and `string_view`
* Lifetime issues
* Passing strings efficiently

---

# 3. Iterators

## 3.1 Iterator Basics

Study:

* What iterators are
* Iterator syntax
* `begin()`
* `end()`
* `cbegin()`
* `cend()`
* `rbegin()`
* `rend()`
* Dereferencing
* Incrementing
* Iterator ranges: `[first, last)`

Example:

```cpp
for (auto it = v.begin(); it != v.end(); ++it) {
    cout << *it << " ";
}
```

---

## 3.2 Iterator Categories

Study:

* Input iterator
* Output iterator
* Forward iterator
* Bidirectional iterator
* Random access iterator
* Contiguous iterator concept, mostly formalized later, but useful for understanding `vector` and `array`

Know which containers support which iterator types:

* `vector`, `array`, `deque`: random access
* `list`, `set`, `map`: bidirectional
* `forward_list`: forward only
* `unordered_*`: forward

---

## 3.3 Iterator Utilities

Study:

* `std::advance`
* `std::distance`
* `std::next`
* `std::prev`
* `std::back_inserter`
* `std::front_inserter`
* `std::inserter`
* `std::istream_iterator`
* `std::ostream_iterator`

---

## 3.4 Iterator Invalidation

Study deeply:

* When `vector` invalidates iterators
* When `deque` invalidates iterators
* When `list` preserves iterators
* When `map` and `set` preserve iterators
* When unordered containers invalidate iterators due to rehashing

This is one of the most important STL topics.

---

# 4. STL Algorithms Library

Header:

```cpp
#include <algorithm>
```

## 4.1 Algorithm Design Principles

Study:

* Algorithms work on iterator ranges
* Range format: `[begin, end)`
* Algorithms do not usually change container size
* Use erase-remove idiom for removal
* Algorithms are generic

---

## 4.2 Non-Modifying Sequence Algorithms

Study:

* `all_of`
* `any_of`
* `none_of`
* `for_each`
* `count`
* `count_if`
* `find`
* `find_if`
* `find_if_not`
* `mismatch`
* `equal`
* `search`
* `find_end`
* `find_first_of`
* `adjacent_find`

Practice:

```cpp
auto it = find(v.begin(), v.end(), 10);
```

---

## 4.3 Modifying Sequence Algorithms

Study:

* `copy`
* `copy_if`
* `copy_n`
* `copy_backward`
* `move`
* `move_backward`
* `fill`
* `fill_n`
* `transform`
* `generate`
* `generate_n`
* `remove`
* `remove_if`
* `replace`
* `replace_if`
* `swap`
* `swap_ranges`
* `reverse`
* `rotate`
* `random_shuffle` deprecated in C++14, removed in C++17
* `shuffle`
* `unique`

Important pattern:

```cpp
v.erase(remove(v.begin(), v.end(), x), v.end());
```

---

## 4.4 Sorting and Ordering Algorithms

Study:

* `sort`
* `stable_sort`
* `partial_sort`
* `partial_sort_copy`
* `nth_element`
* `is_sorted`
* `is_sorted_until`

Important:

```cpp
sort(v.begin(), v.end());
sort(v.begin(), v.end(), greater<int>());
```

Custom sorting:

```cpp
sort(v.begin(), v.end(), [](auto &a, auto &b) {
    return a.second < b.second;
});
```

Use cases:

* Sorting numbers
* Sorting pairs
* Sorting structs
* Sorting by multiple fields
* Top K using `nth_element`

---

## 4.5 Binary Search Algorithms

Use only on sorted ranges.

Study:

* `binary_search`
* `lower_bound`
* `upper_bound`
* `equal_range`

Important:

```cpp
auto it = lower_bound(v.begin(), v.end(), x);
```

Know:

* `lower_bound`: first element `>= x`
* `upper_bound`: first element `> x`
* `equal_range`: range of elements equal to `x`

Use cases:

* Searching sorted arrays
* Counting occurrences
* Coordinate compression
* Range queries

---

## 4.6 Set Algorithms

Use on sorted ranges.

Study:

* `includes`
* `set_union`
* `set_intersection`
* `set_difference`
* `set_symmetric_difference`

Use cases:

* Combining sorted lists
* Comparing sorted collections
* Intersection problems

---

## 4.7 Heap Algorithms

Study:

* `make_heap`
* `push_heap`
* `pop_heap`
* `sort_heap`
* `is_heap`
* `is_heap_until`

Understand:

* How these differ from `priority_queue`
* Max heap by default
* Custom comparator

Practice:

```cpp
make_heap(v.begin(), v.end());
v.push_back(x);
push_heap(v.begin(), v.end());
pop_heap(v.begin(), v.end());
v.pop_back();
```

---

## 4.8 Min/Max Algorithms

Study:

* `min`
* `max`
* `minmax`
* `min_element`
* `max_element`
* `minmax_element`
* `clamp` // C++17

Example:

```cpp
auto it = max_element(v.begin(), v.end());
```

---

## 4.9 Permutation Algorithms

Study:

* `next_permutation`
* `prev_permutation`
* `is_permutation`

Use cases:

* Generating permutations
* Brute force search
* Lexicographic ordering

Example:

```cpp
sort(v.begin(), v.end());
do {
    // process permutation
} while (next_permutation(v.begin(), v.end()));
```

---

## 4.10 Partition Algorithms

Study:

* `partition`
* `stable_partition`
* `is_partitioned`
* `partition_point`
* `partition_copy`

Use cases:

* Separating elements by condition
* Quickselect-like logic
* Rearranging arrays

---

# 5. Numeric Algorithms

Header:

```cpp
#include <numeric>
```

## 5.1 Basic Numeric Algorithms

Study:

* `accumulate`
* `inner_product`
* `partial_sum`
* `adjacent_difference`
* `iota`

Example:

```cpp
int sum = accumulate(v.begin(), v.end(), 0);
```

Important:

```cpp
long long sum = accumulate(v.begin(), v.end(), 0LL);
```

---

## 5.2 C++17 Numeric Algorithms

Study:

* `reduce`
* `transform_reduce`
* `exclusive_scan`
* `inclusive_scan`
* `transform_exclusive_scan`
* `transform_inclusive_scan`
* `gcd`
* `lcm`

Important distinction:

* `accumulate` is ordered
* `reduce` may reorder operations, especially with execution policies

---

# 6. Function Objects and Lambdas

## 6.1 Lambdas

Study:

* Basic lambda syntax
* Capture by value
* Capture by reference
* Generic lambdas
* Mutable lambdas
* Lambdas with STL algorithms

Example:

```cpp
sort(v.begin(), v.end(), [](int a, int b) {
    return a > b;
});
```

---

## 6.2 Standard Function Objects

Header:

```cpp
#include <functional>
```

Study:

* `less`
* `greater`
* `equal_to`
* `not_equal_to`
* `plus`
* `minus`
* `multiplies`
* `divides`
* `modulus`
* `logical_and`
* `logical_or`
* `logical_not`
* `bit_and`
* `bit_or`
* `bit_xor`

Use:

```cpp
sort(v.begin(), v.end(), greater<int>());
```

---

## 6.3 `std::function`

Study:

* Type-erased callable wrapper
* Storing functions, lambdas, functors
* Performance cost
* When to use it

---

## 6.4 `std::bind`

Study:

* Binding arguments
* Placeholders
* Why lambdas are usually preferred

---

# 7. Pair, Tuple, and Structured Bindings

## 7.1 `std::pair`

Study:

* Declaration
* `first`
* `second`
* Pair comparison
* Pair sorting
* Returning two values

Example:

```cpp
pair<int, string> p = {1, "one"};
```

---

## 7.2 `std::tuple`

Study:

* Multiple values
* `get<>()`
* `make_tuple`
* `tie`
* Tuple comparison

Example:

```cpp
auto t = make_tuple(1, "abc", 3.14);
```

---

## 7.3 Structured Bindings, C++17

Study:

```cpp
auto [x, y] = pair<int, int>{1, 2};
```

Use with:

* Pairs
* Tuples
* Maps
* Structs

Example:

```cpp
for (auto &[key, value] : mp) {
    cout << key << " " << value << endl;
}
```

---

# 8. Smart Pointers and Memory Utilities

## 8.1 `std::unique_ptr`

Study:

* Exclusive ownership
* `make_unique`
* Move-only behavior
* Custom deleters

C++14:

```cpp
auto p = make_unique<int>(10);
```

---

## 8.2 `std::shared_ptr`

Study:

* Shared ownership
* Reference counting
* `make_shared`
* Cyclic reference problem

---

## 8.3 `std::weak_ptr`

Study:

* Non-owning reference
* Breaking cycles
* `lock`
* `expired`

---

# 9. Utility Components

## 9.1 `std::optional`, C++17

Study:

* Represents optional value
* `has_value`
* `value`
* `value_or`
* Avoiding sentinel values

Example:

```cpp
optional<int> findValue();
```

---

## 9.2 `std::variant`, C++17

Study:

* Type-safe union
* `holds_alternative`
* `get`
* `get_if`
* `visit`

---

## 9.3 `std::any`, C++17

Study:

* Stores any copyable type
* `any_cast`
* Type safety
* When not to use it

---

## 9.4 `std::string_view`, C++17

Study:

* Non-owning string reference
* Avoiding string copies
* Lifetime risks
* Function parameters

---

## 9.5 `std::filesystem`, C++17

Study:

* Paths
* Directory iteration
* File existence checks
* File size
* Creating directories
* Removing files

Note:

* Some older compilers require special linking for filesystem support.

---

## 9.6 `std::chrono`

Study:

* Clocks
* Durations
* Time points
* Measuring execution time
* `steady_clock`
* `system_clock`
* `high_resolution_clock`

Example:

```cpp
auto start = chrono::steady_clock::now();
```

---

## 9.7 Random Number Library

Header:

```cpp
#include <random>
```

Study:

* `random_device`
* `mt19937`
* `uniform_int_distribution`
* `uniform_real_distribution`
* Shuffling with random engines

Example:

```cpp
mt19937 rng(random_device{}());
shuffle(v.begin(), v.end(), rng);
```

---

# 10. C++14 STL-Relevant Features

Study:

* Generic lambdas
* `auto` return type deduction
* `decltype(auto)`
* `std::make_unique`
* Relaxed constexpr
* Variable templates
* Binary literals
* Digit separators

Important examples:

```cpp
auto add = [](auto a, auto b) {
    return a + b;
};
```

```cpp
auto p = make_unique<int>(42);
```

---

# 11. C++17 STL-Relevant Features

Study:

* Structured bindings
* `if constexpr`
* Inline variables
* Fold expressions
* Class template argument deduction
* `std::optional`
* `std::variant`
* `std::any`
* `std::string_view`
* `std::filesystem`
* Parallel algorithms
* New numeric algorithms
* `std::clamp`
* `std::gcd`
* `std::lcm`

Examples:

```cpp
vector v = {1, 2, 3}; // CTAD
```

```cpp
if constexpr (is_integral_v<T>) {
    // compile-time branch
}
```

---

# 12. Comparators and Custom Ordering

## 12.1 Custom Comparator for `sort`

Study:

```cpp
sort(v.begin(), v.end(), [](const auto &a, const auto &b) {
    return a.score > b.score;
});
```

Topics:

* Strict weak ordering
* Sorting ascending/descending
* Sorting pairs
* Sorting structs/classes
* Multi-key sorting

---

## 12.2 Custom Comparator for `set` and `map`

Study:

```cpp
set<int, greater<int>> s;
```

Custom struct comparator:

```cpp
struct Compare {
    bool operator()(const Node &a, const Node &b) const {
        return a.x < b.x;
    }
};
```

---

## 12.3 Custom Comparator for `priority_queue`

Study:

```cpp
priority_queue<int, vector<int>, greater<int>> pq;
```

For structs:

```cpp
struct Compare {
    bool operator()(const Node &a, const Node &b) {
        return a.cost > b.cost;
    }
};
```

---

# 13. Hashing and Custom Hash Functions

## 13.1 Hashing Basics

Study:

* How `unordered_map` works
* Hash function
* Equality function
* Buckets
* Collisions
* Load factor
* Rehashing

---

## 13.2 Custom Hash for `pair`

Study:

```cpp
struct PairHash {
    size_t operator()(const pair<int, int>& p) const {
        return hash<int>()(p.first) ^ (hash<int>()(p.second) << 1);
    }
};
```

---

## 13.3 Custom Hash for Struct

Study:

```cpp
struct Point {
    int x, y;
};

struct PointHash {
    size_t operator()(const Point &p) const {
        return hash<int>()(p.x) ^ (hash<int>()(p.y) << 1);
    }
};
```

Also study:

* Custom equality operator
* Avoiding poor hash functions
* Reserving unordered containers

---

# 14. Common STL Idioms

## 14.1 Erase-Remove Idiom

```cpp
v.erase(remove(v.begin(), v.end(), x), v.end());
```

Study:

* Why `remove` does not erase
* How container size changes
* `remove_if`

---

## 14.2 Sort and Unique

```cpp
sort(v.begin(), v.end());
v.erase(unique(v.begin(), v.end()), v.end());
```

Use for:

* Removing duplicates from vector
* Coordinate compression

---

## 14.3 Frequency Counting

```cpp
unordered_map<int, int> freq;
for (int x : v) freq[x]++;
```

---

## 14.4 Prefix Sum

```cpp
vector<long long> pref(n + 1);
for (int i = 0; i < n; i++) {
    pref[i + 1] = pref[i] + a[i];
}
```

Also study:

* `partial_sum`
* Range sum queries

---

## 14.5 Coordinate Compression

Study:

* Copy values
* Sort
* Unique
* Use `lower_bound`

Pattern:

```cpp
vector<int> vals = a;
sort(vals.begin(), vals.end());
vals.erase(unique(vals.begin(), vals.end()), vals.end());

int compressed = lower_bound(vals.begin(), vals.end(), x) - vals.begin();
```

---

## 14.6 Two Pointers with STL

Study:

* Sorted arrays
* Sliding windows
* `deque` for window maximum
* Iterators vs indices

---

## 14.7 Using `lower_bound` with Containers

Study:

* `std::lower_bound` on vectors
* `set.lower_bound`
* `map.lower_bound`

Important:

* Use member `lower_bound` for `set` and `map`, not the generic algorithm, because generic `lower_bound` is inefficient on non-random-access iterators.

---

# 15. STL in Competitive Programming

## 15.1 Essential Containers

Study:

* `vector`
* `string`
* `pair`
* `tuple`
* `set`
* `map`
* `unordered_map`
* `priority_queue`
* `deque`
* `stack`
* `queue`
* `bitset`

---

## 15.2 Essential Algorithms

Study:

* `sort`
* `reverse`
* `lower_bound`
* `upper_bound`
* `binary_search`
* `max_element`
* `min_element`
* `accumulate`
* `next_permutation`
* `unique`
* `rotate`
* `nth_element`
* `partial_sum`
* `iota`

---

## 15.3 Common Patterns

Study:

* Frequency maps
* Sorting pairs
* Greedy with priority queue
* BFS with queue
* DFS with stack/vector
* Dijkstra with priority queue
* Sliding window with deque
* Top K with heap
* Deduplication with sort and unique
* Range queries with prefix sums
* Ordered lookup with set/map
* Hash lookup with unordered_map

---

# 16. Complexity Analysis

## 16.1 Big-O for Containers

Study:

* `vector`: random access O(1), push back amortized O(1)
* `deque`: front/back insertion O(1)
* `list`: insertion/erase O(1) with iterator
* `set/map`: insert/find/erase O(log n)
* `unordered_map/unordered_set`: average O(1), worst O(n)
* `priority_queue`: push/pop O(log n), top O(1)

---

## 16.2 Big-O for Algorithms

Study:

* `sort`: O(n log n)
* `stable_sort`: O(n log n)
* `find`: O(n)
* `binary_search`: O(log n)
* `lower_bound`: O(log n) on random access iterators
* `nth_element`: average O(n)
* `make_heap`: O(n)
* `push_heap`: O(log n)
* `pop_heap`: O(log n)
* `next_permutation`: O(n)
* `accumulate`: O(n)

---

# 17. C++17 Parallel Algorithms

Header:

```cpp
#include <execution>
```

Study:

* Execution policies:

```cpp
std::execution::seq
std::execution::par
std::execution::par_unseq
```

Algorithms that may support policies:

* `sort`
* `for_each`
* `transform`
* `reduce`
* `count`
* `find`

Important:

* Not all compilers fully support parallel execution
* Avoid data races
* Operations must be safe for parallel execution
* Results may differ for non-associative operations like floating-point addition

---

# 18. Advanced STL Topics

## 18.1 Allocators

Study:

* What allocators do
* Default allocator
* Custom allocators
* Allocator-aware containers
* Memory pools

Useful if:

* You work on high-performance systems
* You need custom memory management

---

## 18.2 Traits

Study:

* `type_traits`
* `iterator_traits`
* `char_traits`
* `numeric_limits`

Common type traits:

```cpp
is_same
is_integral
is_floating_point
is_pointer
enable_if
conditional
remove_reference
decay
```

C++17 helpers:

```cpp
is_same_v<T, U>
is_integral_v<T>
```

---

## 18.3 SFINAE and `enable_if`

Study:

* Template substitution
* Function overload control
* Type constraints before concepts
* Relevance to STL design

---

## 18.4 `if constexpr`, C++17

Study:

* Compile-time branching
* Replacing some SFINAE patterns
* Writing generic functions more cleanly

---

## 18.5 Move Iterators

Study:

* `make_move_iterator`
* Moving ranges
* Difference between copy and move algorithms

---

# 19. Debugging STL Code

Study:

* Iterator out of range errors
* Dereferencing `end()`
* Invalidated iterators
* Off-by-one errors
* Using `at()` for bounds checking
* Debug STL modes
* Printing containers
* Avoiding accidental copies
* Const correctness

Common mistakes:

```cpp
for (int i = 0; i <= v.size(); i++) // wrong
```

```cpp
auto it = v.end();
cout << *it; // wrong
```

---

# 20. Suggested Study Order

## Phase 1: Core STL Basics

1. `vector`
2. `string`
3. Iterators
4. Range-based loops
5. `sort`
6. `find`
7. `reverse`
8. `min_element`, `max_element`
9. `accumulate`
10. `pair`

## Phase 2: Competitive Programming Essentials

1. `set`
2. `map`
3. `unordered_map`
4. `unordered_set`
5. `stack`
6. `queue`
7. `priority_queue`
8. `lower_bound`
9. `upper_bound`
10. `next_permutation`

## Phase 3: Intermediate STL

1. Custom comparators
2. Lambdas
3. `deque`
4. `list`
5. `tuple`
6. `bitset`
7. `unique`
8. Erase-remove idiom
9. `nth_element`
10. Heap algorithms

## Phase 4: C++14 and C++17 Features

1. Generic lambdas
2. `make_unique`
3. Structured bindings
4. `if constexpr`
5. `optional`
6. `variant`
7. `string_view`
8. `filesystem`
9. Parallel algorithms
10. New numeric algorithms

## Phase 5: Advanced STL

1. Iterator invalidation
2. Allocators
3. Type traits
4. SFINAE
5. Custom hash functions
6. Move iterators
7. Performance tuning
8. Parallel execution safety

---

# 21. Practice Problem Categories

To master STL, practice problems involving:

## Containers

* Dynamic array operations
* Frequency counting
* Duplicate removal
* Ordered set queries
* Hash map lookup
* Priority queue scheduling

## Algorithms

* Sorting custom objects
* Binary search
* Permutations
* Heap operations
* Partitioning
* Prefix sums
* Range transformation

## Common Problem Types

* Two sum
* Top K frequent elements
* Merge intervals
* Kth largest element
* Longest substring without repeating characters
* Sliding window maximum
* Meeting rooms
* Dijkstra’s algorithm
* Word frequency counter
* Coordinate compression
* Next permutation
* Median from data stream

---

# 22. Must-Know STL Snippets

## Sort vector

```cpp
sort(v.begin(), v.end());
```

## Sort descending

```cpp
sort(v.begin(), v.end(), greater<int>());
```

## Remove value from vector

```cpp
v.erase(remove(v.begin(), v.end(), x), v.end());
```

## Remove duplicates

```cpp
sort(v.begin(), v.end());
v.erase(unique(v.begin(), v.end()), v.end());
```

## Frequency map

```cpp
unordered_map<int, int> freq;
for (int x : v) freq[x]++;
```

## Lower bound index

```cpp
int idx = lower_bound(v.begin(), v.end(), x) - v.begin();
```

## Min heap

```cpp
priority_queue<int, vector<int>, greater<int>> pq;
```

## Iterate map, C++17

```cpp
for (auto &[key, value] : mp) {
    cout << key << " " << value << endl;
}
```

## Sum with `long long`

```cpp
long long sum = accumulate(v.begin(), v.end(), 0LL);
```

## Generate sequence

```cpp
iota(v.begin(), v.end(), 1);
```

---

# 23. Final Checklist

You should be comfortable with:

* Choosing the right STL container
* Understanding iterator categories
* Knowing iterator invalidation rules
* Using STL algorithms correctly
* Writing custom comparators
* Using lambdas with algorithms
* Using `map`, `set`, `unordered_map`, and `priority_queue`
* Applying binary search with `lower_bound` and `upper_bound`
* Using C++14 generic lambdas
* Using C++14 `make_unique`
* Using C++17 structured bindings
* Using C++17 `optional`, `variant`, `any`, and `string_view`
* Understanding complexity of STL operations
* Avoiding common STL mistakes
* Writing clean and efficient modern C++ code
