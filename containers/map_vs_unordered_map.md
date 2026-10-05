# API Comparison: `std::map` vs. `std::unordered_map`

While `std::map` and `std::unordered_map` both store key-value pairs with unique keys in C++, their underlying data structures lead to distinct differences in their APIs, iterator capabilities, key type requirements, and performance guarantees.

---

## 1. Core Data Structures & Type Requirements

| Feature | `std::map` | `std::unordered_map` |
| :--- | :--- | :--- |
| **Header** | `<map>` | `<unordered_map>` |
| **Data Structure** | Red-Black Tree (Balanced Binary Search Tree) | Hash Table (Array of Buckets) |
| **Element Order** | Sorted by key (ascending by default) | Arbitrary / Unordered |
| **Key Type Requirements** | Strict weak ordering: `std::less<Key>` or custom `<` operator | Hash function (`std::hash<Key>`) & equality predicate (`std::equal_to<Key>` or `==`) |

---

## 2. Overview of API Differences

| API / Feature | `std::map` | `std::unordered_map` |
| :--- | :--- | :--- |
| **Iterator Category** | **Bidirectional Iterator** (`++`, `--`) | **Forward Iterator** (`++` only) |
| **Reverse Iterators** | Supported (`rbegin()`, `rend()`) | Not supported |
| **Iterator Invalidation** | **Never invalidates** existing iterators on insert/delete (unless deleting the specific element) | Insertion **may invalidate** iterators if a rehash occurs |
| **Range Queries** | Supported (`lower_bound`, `upper_bound`) | Not supported |
| **Bucket Inspection** | Not applicable | Supported (`bucket_count`, `bucket_size`, `bucket`, etc.) |
| **Lookup Time Complexity** | $O(\log n)$ (Logarithmic) | $O(1)$ average, $O(n)$ worst-case (Linear) |

---

## 3. Container-Specific APIs

### A. Operations Unique to `std::map` (Order-Dependent)

Because `std::map` maintains keys in a sorted sequence, it provides specialized lookup methods that exploit element ordering:

* **`lower_bound(key)`**: Returns an iterator to the first element whose key is **not less** than `key` (i.e., $\ge \text{key}$).
* **`upper_bound(key)`**: Returns an iterator to the first element whose key is **greater** than `key` (i.e., $> \text{key}$).
* **`equal_range(key)`**: Returns a `std::pair` of iterators representing the range `[lower_bound, upper_bound)`.
* **Reverse Iteration**: Exposes `rbegin()`, `rend()`, `crbegin()`, and `crend()`.

---

### B. Operations Unique to `std::unordered_map` (Hash-Table-Dependent)

Because `std::unordered_map` uses an internal array of buckets, it exposes APIs to inspect and tune the state of the hash table:

#### Bucket Interface
* **`bucket_count()`**: Returns the current number of buckets in the hash table.
* **`max_bucket_count()`**: Returns the maximum number of buckets the container can hold.
* **`bucket_size(n)`**: Returns the number of elements in bucket index `n`.
* **`bucket(key)`**: Returns the bucket index where the given `key` is assigned.

#### Hash & Load Factor Control
* **`load_factor()`**: Returns the average number of elements per bucket (`size() / bucket_count()`).
* **`max_load_factor()`**: Gets or sets the load factor threshold before a rehash occurs.
* **`rehash(n)`**: Sets the number of buckets to at least `n` and rehashes the entire container.
* **`reserve(n)`**: Sets the bucket count to accommodate at least `n` elements without exceeding the maximum load factor.
* **`hash_function()` & `key_eq()`**: Returns the hashing and key equality functors used by the container.

---

## 4. Shared APIs with Different Behavior

Both containers share common interface methods, but their underlying performance guarantees differ:

```cpp
// Common operations available on both containers:
map[key];              // Access or insert
map.at(key);           // Access with bounds checking
map.find(key);         // Search for an element
map.insert({...});     // Insert key-value pair
map.emplace(key, val); // Construct in-place
map.erase(key);        // Remove element by key
```

* **Lookup Operations (`operator[]`, `at()`, `find()`)**:
  * `std::map`: Performs a tree traversal in **$O(\log n)$** time.
  * `std::unordered_map`: Computes key hash and performs bucket lookup in **$O(1)$** average time ($O(n)$ if many hash collisions occur).

* **Insertion & Deletion (`insert()`, `emplace()`, `erase()`)**:
  * `std::map`: Performs tree rebalancing in **$O(\log n)$** time. Iterators to existing elements remain valid.
  * `std::unordered_map`: Inserts in **$O(1)$** average time. If insertion causes the load factor to exceed `max_load_factor()`, a rehash occurs, which invalidates all existing iterators.
  * 
