# The C++ Spaceship Operator (`<=>`): A Comprehensive Guide for C++20 and C++23

Introduced in C++20 and further refined in C++23, the **three-way comparison operator**—popularly known as the **spaceship operator** (`<=>`)—fundamentally changes how object comparison is implemented and handled in Modern C++. 

This guide details the mechanics of the spaceship operator, why and when to use it, how it interacts with various types, and the auxiliary utilities provided by the C++ Standard Library.

---

## 1. Why Use the Spaceship Operator?

Prior to C++20, implementing complete ordering for a custom class required writing up to six relational operators (`==`, `!=`, `<`, `<=`, `>`, `>=`). This was tedious, error-prone, and inflated boilerplate code.

```cpp
// Pre-C++20: Writing relational operators manually
struct LegacyPoint {
    int x;
    int y;

    bool operator==(const LegacyPoint& rhs) const { return x == rhs.x && y == rhs.y; }
    bool operator!=(const LegacyPoint& rhs) const { return !(*this == rhs); }
    bool operator<(const LegacyPoint& rhs) const {
        if (x != rhs.x) return x < rhs.x;
        return y < rhs.y;
    }
    bool operator<=(const LegacyPoint& rhs) const { return !(rhs < *this); }
    bool operator>(const LegacyPoint& rhs) const { return rhs < *this; }
    bool operator>=(const LegacyPoint& rhs) const { return !(*this < rhs); }
};
```

### Key Advantages of `<=>`

1. **Boilerplate Elimination**: Defining `operator<=>` (along with `operator==` in some cases) automatically enables all six comparison operators (`==`, `!=`, `<`, `<=`, `>`, `>=`).
2. **Compiler Generation**: The compiler can auto-generate (`= default`) memberwise comparison.
3. **Optimized Comparisons**: Traditional three-way comparisons often required multiple branch checks (e.g., `<` followed by `==`). A single call to `<=>` performs one check and returns a comparison category object.
4. **Consistency**: Eliminates human error where `<` and `==` logic might diverge or violate mathematical ordering properties.

---

## 2. How to Use It

To use the spaceship operator and its associated comparison categories, include the standard `<compare>` header.

### 2.1 Defaulted Comparison (`= default`)

The simplest way to use `<=>` is to request compiler generation:

```cpp
#include <compare>
#include <string>

struct Employee {
    int id;
    std::string name;
    double salary;

    // Compiler generates memberwise <=> and memberwise ==
    auto operator<=>(const Employee&) const = default;
};
```

When you define `auto operator<=>(...) const = default;`, the compiler:
- Performs lexicographical (member-by-member) comparison in the order members are declared.
- Automatically generates `operator==` (which defaults to memberwise equality for efficiency).

#### Usage Example:
```cpp
Employee e1{101, "Alice", 75000.0};
Employee e2{102, "Bob", 80000.0};

bool is_less = (e1 < e2);   // True (101 < 102)
bool is_equal = (e1 == e2); // False
```

### 2.2 Custom Implementation

If memberwise lexicographical comparison does not meet your needs, you can implement `operator<=>` manually.

```cpp
#include <compare>
#include <cmath>

class Vector2D {
public:
    double x;
    double y;

    double magnitude_sq() const { return x * x + y * y; }

    // Custom three-way comparison based on magnitude
    std::partial_ordering operator<=>(const Vector2D& rhs) const {
        return magnitude_sq() <=> rhs.magnitude_sq();
    }

    // Note: Custom operator<=> does NOT auto-generate operator==
    bool operator==(const Vector2D& rhs) const {
        return magnitude_sq() == rhs.magnitude_sq();
    }
};
```

*Note: When you provide a custom `operator<=>`, you must also explicitly define `operator==` if you require custom equality logic, as the compiler only auto-generates `operator==` when `operator<=>` is defaulted.*

---

## 3. Comparison Categories in C++

The return type of `operator<=>` is not a `bool` or an `int`; it is one of three comparison category types defined in `<compare>`:

| Comparison Category | Equivalence Meaning | Substitutability | Example Types |
| :--- | :--- | :--- | :--- |
| `std::strong_ordering` | Equals means identical | Absolute (`a == b` $\Rightarrow f(a) == f(b)$) | `int`, `std::string`, `std::vector` |
| `std::weak_ordering` | Equivalent, but may not be identical | Case-insensitive strings, case-folded data | Custom case-insensitive string |
| `std::partial_ordering` | Some values are unordered | Non-comparable values exist | `float`, `double` (`NaN` is unordered) |

### 3.1 `std::strong_ordering`

Provides complete ordering where equal values are completely indistinguishable in behavior or value.
- Values: `strong_ordering::less`, `strong_ordering::equal`, `strong_ordering::equivalent`, `strong_ordering::greater`.

### 3.2 `std::weak_ordering`

Provides complete ordering where two values can be equivalent without being identical.
- Values: `weak_ordering::less`, `weak_ordering::equivalent`, `weak_ordering::greater`.

### 3.3 `std::partial_ordering`

Provides ordering where some pairs of values cannot be compared (e.g., IEEE floating-point `NaN`).
- Values: `partial_ordering::less`, `partial_ordering::equivalent`, `partial_ordering::greater`, `partial_ordering::unordered`.

---

## 4. Behavior across Different Types in C++23

### 4.1 Fundamental Types

- **Integers and Booleans**: Yield `std::strong_ordering`.
- **Pointers**: Yield `std::strong_ordering`.
- **Floating-Point Types (`float`, `double`, `long double`)**: Yield `std::partial_ordering` due to `NaN` values.

```cpp
#include <compare>
#include <iostream>

void check_floats(double a, double b) {
    auto res = a <=> b;
    if (res == std::partial_ordering::unordered) {
        std::cout << "Unordered comparison (likely NaN)\n";
    } else if (res < 0) {
        std::cout << "a < b\n";
    }
}
```

### 4.2 Standard Containers and Types in C++20/C++23

Standard types overload `<=>` to adopt the strongest category supported by their elements:
- `std::string`, `std::vector<int>` return `std::strong_ordering`.
- `std::vector<double>` returns `std::partial_ordering` (inherited from `double`).
- `std::optional<T>` and `std::pair<T1, T2>` deduce their category based on contained types.

### 4.3 C++23 Enhancements

C++23 added and refined `<=>` support across standard library utilities:
1. **`std::optional`**: Added full constrained `<=>` support for mixed comparisons (e.g., `optional<T> <=> U`).
2. **`std::unique_ptr`**: Syntactical improvements for comparisons against `std::nullptr_t`.
3. **`std::reference_wrapper`**: Enhanced comparison support in standard headers.

---

## 5. Standard Utilities in `<compare>`

The standard library provides helper templates and concepts to interact with three-way comparisons.

### 5.1 Helper Functions

When writing generic code, prefer using standard helper algorithms over calling `<=>` directly:

- **`std::compare_three_way`**: Function object wrapping `<=>`.
- **`std::compare_strong_order_fallback(a, b)`**: Performs a strong ordering comparison, falling back to `<` and `==` if `<=>` is unavailable.
- **`std::compare_weak_order_fallback(a, b)`**: Performs weak ordering comparison with fallback.
- **`std::compare_partial_order_fallback(a, b)`**: Performs partial ordering comparison with fallback.

```cpp
#include <compare>

template <typename T>
bool generic_is_less(const T& a, const T& b) {
    // Works for types implementing <=> OR types implementing older < and ==
    return std::compare_weak_order_fallback(a, b) < 0;
}
```

### 5.2 Concepts

C++20/C++23 provide constraints in `<compare>` to concepts-check ordering capabilities:

- `std::three_way_comparable<T>`
- `std::three_way_comparable_with<T, U>`

```cpp
#include <compare>

template <std::three_way_comparable T>
void sort_custom(std::vector<T>& vec) {
    // Ensured at compile-time that T supports <=>
}
```

---

## 6. When to Use and When NOT to Use

### When to Use `<=>`

1. **Default Data Structures**: Any `struct` or `class` representing a aggregate record or value object where memberwise comparison is natural.
2. **Custom Numeric / Key Types**: Custom types meant to be used as keys in `std::map`, elements in `std::set`, or sorted in `std::vector`.
3. **Generic Library Design**: When building templates where elements need consistent, efficient ordering.

### When NOT to Use `<=>`

1. **Types Without Logical Ordering**: Types representing resources (e.g., `std::mutex`, `std::thread`, database connections) should not be compared.
2. **Non-Lexicographical Defaults**: Avoid `= default` if member declaration order does not match the desired sorting hierarchy.
3. **Performance-Critical Floating-Point Aggregates**: If you need strict `strong_ordering` for map keys containing floating-point numbers, `auto operator<=> = default` will produce `std::partial_ordering`, which cannot directly satisfy some strong-ordering constraints without adaptation.

---

## Summary Checklist

- [x] `#include <compare>` to access comparison types and utilities.
- [x] Use `auto operator<=>(const ClassName&) const = default;` for standard value types.
- [x] `auto operator<=> = default` auto-generates `operator==`.
- [x] A **custom** `operator<=>` does **not** auto-generate `operator==`. Implement `operator==` explicitly.
- [x] Use fallback utilities like `std::compare_strong_order_fallback` in template code for backward compatibility.
```

eof
