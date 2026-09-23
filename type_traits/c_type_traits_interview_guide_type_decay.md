# C++ Metaprogramming Interview Guide: Implementing `std::decay`

## Problem Prompt

**Title:** Implementing a Custom `std::decay`

**Difficulty:** Advanced (Senior C++ Engineer / Metaprogramming Focus)

### Problem Statement

In C++, function arguments passed by value undergo standard conversions known as *decay*:

* References are stripped.
* Array types decay into pointers to their element types.
* Function types decay into function pointers.
* For all other types, top-level `const` and `volatile` qualifiers are removed.

Your task is to implement a custom type trait `my_decay<T>` and its companion template alias `my_decay_t<T>` without relying on `std::decay`.

### Requirements

1. **Reference Stripping**: Must handle lvalue references (`&`) and rvalue references (`&&`).
2. **Array Decay**: Types like `T[N]` or `T[]` must decay to `T*` (or `const T*` if the elements are const-qualified).
3. **Function Decay**: Function types such as `R(Args...)` must decay to `R(*)(Args...)`.
4. **Value Types**: All other non-array, non-function types must strip top-level cv-qualifiers (e.g., `const int` $\to$ `int`).
5. **Ordering**: Ensure combinations like references to arrays (e.g., `int(&)[5]`) or references to functions (e.g., `void(&)(int)`) decay correctly to pointers.

## The Complete Solution

```cpp
#include <type_traits>

namespace detail {

// --- Step 1: Strip references ---
template <typename T>
struct remove_reference {
    using type = T;
};

template <typename T>
struct remove_reference<T&> {
    using type = T;
};

template <typename T>
struct remove_reference<T&&> {
    using type = T;
};

template <typename T>
using remove_reference_t = typename remove_reference<T>::type;

// --- Step 2: Primary selector and partial specializations ---

// Primary template: Fallback for scalar/class types (strip top-level cv-qualifiers)
template <typename U, 
          bool IsArray = std::is_array_v<U>, 
          bool IsFunc  = std::is_function_v<U>>
struct decay_selector {
    using type = std::remove_cv_t<U>;
};

// Specialization 1: Array types -> pointer to element type
template <typename U>
struct decay_selector<U, true, false> {
    using type = std::remove_extent_t<U>*;
};

// Specialization 2: Function types -> function pointer
template <typename U>
struct decay_selector<U, false, true> {
    using type = std::add_pointer_t<U>;
};

} // namespace detail

// --- Public API ---

template <typename T>
struct my_decay {
    using type = typename detail::decay_selector<detail::remove_reference_t<T>>::type;
};

template <typename T>
using my_decay_t = typename my_decay<T>::type;
```

## Detailed Technical Explanation

### 1. Order of Operations: Why References Must Be Stripped First

A common pitfall in candidate solutions is attempting to branch on `is_array` or `is_function` directly on `T`:

```cpp
// Flawed logic:
template <typename T>
struct bad_decay {
    // If T is int(&)[5], std::is_array_v<T> evaluates to FALSE!
    // Instead, std::is_reference_v<T> is true.
};
```

In the C++ type system:
* `int[5]` is an array type.
* `int(&)[5]` is a **reference type** (specifically, an lvalue reference to an array of 5 integers).

If we query `std::is_array<int(&)[5]>`, it inherits from `std::false_type`. Therefore, the transformation pipeline **must** be sequential:

```
Input T ---> [ remove_reference ] ---> U
                                       |
       +-------------------------------+-------------------------------+
       |                               |                               |
  Is Array?                       Is Function?                   Other Type
       |                               |                               |
       v                               v                               v
remove_extent_t<U>*             add_pointer_t<U>                remove_cv_t<U>
```

In mathematical notation:

$$
T \xrightarrow{\text{remove\_reference}} U \implies \begin{cases} 
\text{Array:} & \text{remove\_extent\_t}\langle U \rangle^* \\ 
\text{Function:} & \text{add\_pointer\_t}\langle U \rangle \\ 
\text{Other:} & \text{remove\_cv\_t}\langle U \rangle 
\end{cases}
$$

### 2. Array Extent vs. Pointer Conversion

When an array decays, only the *first* extent is converted to a pointer:

* 1D Array: `int[5]` $\to$ `int*`
* Multidimensional Array: `int[5][10]` $\to$ `int(*)[10]`

`std::remove_extent_t<U>` removes only the outermost dimension:
* `std::remove_extent_t<int[5][10]>` is `int[10]`.
* Appending `*` results in `int(*)[10]`, which matches the exact behavior of language-level array-to-pointer decay.

Furthermore, `remove_extent` preserves qualifiers on the element itself:
* `const int[3]` has extent removed to yield `const int`.
* Adding `*` produces `const int*` (pointer to `const int`).

### 3. Function Pointers

A function signature cannot exist as a standalone runtime value; it exists as code in memory. In C and C++, passing a function name to a parameter expecting a value causes the function to decay into a function pointer:

$$
\text{void}(\text{int}, \text{double}) \xrightarrow{\text{decay}} \text{void}(*)(\text{int}, \text{double})
$$

Applying `std::add_pointer_t<U>` directly to a function type transforms the function signature into its corresponding pointer type.

### 4. Non-Array, Non-Function Types

For normal types (integers, pointers, structs, etc.), references have already been stripped. Decay only strips top-level `const` and `volatile` qualifiers:

* `const int` $\to$ `int`
* `int * const` (const pointer to int) $\to$ `int*` (pointer to int)
* `const int*` (pointer to const int) $\to$ `const int*` (the `const` is low-level, so it remains)

`std::remove_cv_t<U>` handles this by stripping top-level qualifiers from $U$.

## Verification and Test Cases

Below is a test suite using `static_assert` to validate all standard decay behaviors at compile time:

```cpp
#include <type_traits>

void test_decay() {
    // 1. Primitive scalars & qualifiers
    static_assert(std::is_same_v<my_decay_t<int>, int>);
    static_assert(std::is_same_v<my_decay_t<const int>, int>);
    static_assert(std::is_same_v<my_decay_t<volatile int>, int>);
    static_assert(std::is_same_v<my_decay_t<const volatile int>, int>);

    // 2. References
    static_assert(std::is_same_v<my_decay_t<int&>, int>);
    static_assert(std::is_same_v<my_decay_t<const int&>, int>);
    static_assert(std::is_same_v<my_decay_t<int&&>, int>);

    // 3. Arrays (bounded and unbounded)
    static_assert(std::is_same_v<my_decay_t<int[5]>, int*>);
    static_assert(std::is_same_v<my_decay_t<const int[5]>, const int*>);
    static_assert(std::is_same_v<my_decay_t<int[]>, int*>);
    static_assert(std::is_same_v<my_decay_t<int(&)[5]>, int*>);
    static_assert(std::is_same_v<my_decay_t<int[5][10]>, int(*)[10]>);

    // 4. Functions
    using Func = void(int, double);
    static_assert(std::is_same_v<my_decay_t<Func>, void(*)(int, double)>);
    static_assert(std::is_same_v<my_decay_t<Func&>, void(*)(int, double)>);

    // 5. Low-level vs high-level const
    static_assert(std::is_same_v<my_decay_t<int* const>, int*>);
    static_assert(std::is_same_v<my_decay_t<const int*>, const int*>);
}
```

## Interviewer Follow-Up Questions

1. **How would you implement `remove_extent` from scratch?**
   * *Answer*: Using partial template specialization pattern matching `T[N]` and `T[]`.

2. **How does `std::is_function` differentiate between a class type and a function type?**
   * *Answer*: A type `T` is a function if and only if `!std::is_reference_v<T>` and `!std::is_const_v<const T>` (since function types cannot be `const`-qualified at the top level).

3. **Where is `std::decay` used in the Standard Library?**
   * *Answer*: In functions that take arguments by forwarding reference but store them by value, such as `std::make_pair`, `std::make_tuple`, and `std::thread` constructors.
