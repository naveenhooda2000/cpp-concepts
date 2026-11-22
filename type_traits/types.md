# Comprehensive Guide to the C++ Type System

C++ is a strongly typed, statically typed programming language. Every entity—including variables, expressions, and objects—has a specific type that is evaluated and verified by the compiler before runtime execution. 

---

## 1. Mental Model: The C++ Type System Hierarchy

The tree structure below maps out how types are organized in C++ from fundamental building blocks to user-defined structures.

```text
C++ Types
│
├── 1. Fundamental Types (Built-in / Primitive)
│   ├── void (Incomplete type representing no value)
│   ├── std::nullptr_t (Type of literal nullptr)
│   │
│   └── Arithmetic Types
│       ├── Floating-Point Types
│       │   ├── float (Single precision, typically 32-bit)
│       │   ├── double (Double precision, typically 64-bit)
│       │   ├── long double (Extended precision, 80-bit or 128-bit)
│       │   └── Fixed-width / Extended (C++23: std::float32_t, std::float64_t, std::bfloat16_t, etc.)
│       │
│       └── Integral Types
│           ├── bool (Boolean value: true or false)
│           │
│           ├── Character Types
│           │   ├── Narrow: char, signed char, unsigned char
│           │   └── Wide / Unicode: wchar_t, char8_t (C++20), char16_t, char32_t
│           │
│           └── Integer Types
│               ├── Signed: signed char, short, int, long, long long
│               ├── Unsigned: unsigned char, unsigned short, unsigned int, unsigned long, unsigned long long
│               └── Fixed-Width Aliases (<cstdint>): int8_t, int32_t, uint64_t, size_t, uintptr_t, etc.
│
└── 2. Compound Types (Derived / User-Defined)
    ├── Reference Types
    │   ├── Lvalue Reference (T&)
    │   └── Rvalue Reference (T&&)
    │
    ├── Pointer Types
    │   ├── Raw Object Pointer (T*)
    │   ├── Pointer-to-void (void*)
    │   └── Function Pointer (R (*)(Args...))
    │
    ├── Pointer-to-Member Types
    │   ├── Pointer to Data Member (T C::*)
    │   └── Pointer to Member Function (R (C::*)(Args...))
    │
    ├── Array Types
    │   ├── Fixed-Size / Bound Array (T[N])
    │   └── Unbounded Array (T[])
    │
    ├── Function Types (R(Args...))
    │
    ├── Enumeration Types
    │   ├── Unscoped Enum (enum Color { Red, Green })
    │   └── Scoped Enum / Enum Class (enum class State { Idle, Running })
    │
    └── Class / Struct / Union Types
        ├── struct (Default public members and inheritance)
        ├── class (Default private members and inheritance)
        └── union (Shares overlapping memory for members)
```

---

## 2. Fundamental Types

Fundamental types are built directly into the language and correspond closely to hardware-level capabilities.

```text
                  ┌─── void
                  ├─── std::nullptr_t
Fundamental Types ┤
                  │                  ┌─── Floating-Point Types
                  └─── Arithmetic ───┤
                                     └─── Integral Types (bool, char*, int*)
```

### 2.1 Void (`void`)
* Represents an empty set of values or an incomplete type.
* Cannot be instantiated directly as a variable (e.g., `void x;` is invalid).
* **Common uses**:
  * Function return type indicating no value is returned.
  * Generic untyped memory address pointer (`void*`).
  * Explicitly empty function parameter lists (legacy C style).

### 2.2 Null Pointer Type (`std::nullptr_t`)
* The distinct type of the literal `nullptr`.
* Resolves overload ambiguities between integer `0` (or `NULL`) and actual pointer types.
* Implicitly convertible to any raw pointer type (`T*`) or pointer-to-member type.

### 2.3 Floating-Point Types
Floating-point types represent real numbers using fractional notation, typically adhering to the IEEE 754 standard.

| Type | Typical Size | Precision | Standard / Features |
| :--- | :--- | :--- | :--- |
| `float` | 32 bits (4 bytes) | ~6–9 decimal digits | Single precision |
| `double` | 64 bits (8 bytes) | ~15–17 decimal digits | Double precision (default for floating literals) |
| `long double` | 80 / 128 bits | ~19–34 decimal digits | Architecture-dependent extended precision |
| `std::float16_t` | 16 bits | 3–4 decimal digits | IEEE half-precision (C++23 standard `<stdfloat>`) |
| `std::float32_t` | 32 bits | ~7 decimal digits | Fixed-width single precision (C++23) |
| `std::float64_t` | 64 bits | ~16 decimal digits | Fixed-width double precision (C++23) |
| `std::float128_t`| 128 bits | ~34 decimal digits | Fixed-width quad precision (C++23) |
| `std::bfloat16_t` | 16 bits | ~3 decimal digits | Brain Floating Point for ML/AI hardware (C++23) |

### 2.4 Integral Types

#### Boolean
* `bool`: Stores logical boolean values (`true` or `false`).
* Machine size is typically 1 byte for direct addressing capability.

#### Character Types
C++ distinguishes between several distinct character types based on encoding standard and width:

* **Narrow Characters**:
  * `char`: Primary character type. Signedness depends on implementation/compiler flags.
  * `signed char`: Explicitly signed 8-bit character integer.
  * `unsigned char`: Explicitly unsigned 8-bit character integer (frequently used for raw byte inspection).
* **Wide & Unicode Characters**:
  * `wchar_t`: Wide character type (16 bits on Windows/MSVC, 32 bits on Linux/GCC).
  * `char8_t`: UTF-8 character representation (C++20, distinct type from `unsigned char`).
  * `char16_t`: UTF-16 character representation (introduced in C++11).
  * `char32_t`: UTF-32 character representation (introduced in C++11).

#### Integer Types
Integers are available in both **signed** (two's complement in C++20 standard) and **unsigned** variations.

```cpp
// Minimum width guarantees by C++ standard:
signed char        // At least 8 bits
short int          // At least 16 bits
int                // At least 16 bits (typically 32 bits on modern platforms)
long int           // At least 32 bits (32 bits on 64-bit Windows, 64 bits on 64-bit Linux)
long long int      // At least 64 bits (C++11)
```

#### Fixed-Width Integer Types (`<cstdint>`)
To guarantee exact integer sizing across cross-platform code, the `<cstdint>` header provides aliases:

* **Exact-width**: `int8_t`, `int16_t`, `int32_t`, `int64_t`, `uint8_t`, `uint16_t`, `uint32_t`, `uint64_t`
* **Pointer-sized**: `intptr_t`, `uintptr_t` (integers guaranteed to be large enough to hold a pointer)
* **Object sizing**: `std::size_t` (unsigned integer type returned by `sizeof`), `std::ptrdiff_t` (signed integer result of pointer subtraction)

---

## 3. Compound & Derived Types

Compound types are constructed by combining fundamental types using language syntax mechanisms.

### 3.1 Reference Types
References act as non-null aliases to existing memory locations.

* **Lvalue Reference (`T&`)**:
  * Binds to lvalues (named objects, non-temporary expressions).
  * `const T&` can bind to both lvalues and temporaries (rvalues), extending temporary lifetime.
* **Rvalue Reference (`T&&`)**:
  * Introduced in C++11 to support **move semantics** and **perfect forwarding**.
  * Binds exclusively to rvalues (temporaries, values returning from expressions).
  * When combined with template type deduction (`auto&&` or `T&&`), acts as a **forwarding reference** (or universal reference).

### 3.2 Pointer Types
Pointers hold the memory address of an object, function, or incomplete type.

```cpp
int x = 42;
int* ptr = &x;                  // Raw object pointer
void* vptr = ptr;               // Generic pointer (erases type information)
int (*funcPtr)(int) = &myFunc;  // Function pointer
```

### 3.3 Pointer-to-Member Types
Unlike regular pointers, pointer-to-members do not store absolute memory addresses. Instead, they store a relative offset into an instance of a class or struct.

```cpp
struct Point {
    int x;
    int y;
    void print() {}
};

// Pointer to data member
int Point::* pData = &Point::x;

// Pointer to member function
void (Point::* pFunc)() = &Point::print;

Point pt{10, 20};
pt.*pData = 15;        // Accessing via object instance
(pt.*pFunc)();         // Invoking member function via pointer
```

### 3.4 Array Types
Contiguous sequence of homogeneous elements stored sequentially in memory.

* **Fixed-size array (`T[N]`)**: Exact bounds specified at compile time.
* **Unbounded array (`T[]`)**: Used in parameter declarations, forward declarations, or extern references. Decay automatically to pointers (`T*`) when passed into functions.

### 3.5 Enumeration Types
Enumerations define a distinct integral type constrained to a set of named constants.

```cpp
// Unscoped Enum (Legacy C-style)
// Names leak into outer scope; implicitly converts to int
enum Color { Red, Green, Blue };

// Scoped Enum / Enum Class (Modern C++11)
// Strongly typed; requires scope resolution; no implicit int conversion
enum class TrafficLight : uint8_t { Red, Yellow, Green };

TrafficLight signal = TrafficLight::Red;
```

### 3.6 Class, Struct, and Union Types

```text
User-Defined Aggregates
│
├── struct  ─── Members & inheritance are public by default
├── class   ─── Members & inheritance are private by default
└── union   ─── All data members share the same starting address space
```

* **`struct`**: Identical to `class`, except default member visibility and inheritance access specifier are `public`.
* **`class`**: Encapsulates data and methods. Default member visibility and inheritance access specifier are `private`.
* **`union`**: A structure where only one data member can be active at any given time. The total allocation size corresponds to the size of its largest member.

---

## 4. Standard Library Type Categories & Metaprogramming Traits

The C++ Standard Library (`<type_traits>`) defines higher-level formal classifications that characterize object behavior during compilation.

```text
Standard Traits Classification Overview:

                  ┌─── Scalar Types (Arithmetic, Pointers, Enums, nullptr_t)
                  ├─── Trivial Types (Trivial default ctor, copy, move, dtor)
Standard Categories ┤
                  ├─── Standard-Layout Types (C-compatible memory layout)
                  ├─── Trivially Copyable Types (Safe for std::memcpy)
                  └─── Literal Types (Usable in compile-time constexpr)
```

| Type Category | Type Trait (`std::`) | Characteristics & Purpose |
| :--- | :--- | :--- |
| **Scalar Types** | `is_scalar<T>` | Represents primitive values: arithmetic types, pointers, pointer-to-members, enumerations, and `std::nullptr_t`. Fits inside CPU registers. |
| **Trivial Types** | `is_trivial<T>` | Has a trivial default constructor and is trivially copyable. Memory allocation needs no custom initialization logic. |
| **Standard-Layout** | `is_standard_layout<T>` | Has C-compatible binary memory layout. No virtual functions, no virtual base classes, and uniform member access control. Safe for C interop. |
| **Trivially Copyable**| `is_trivially_copyable<T>` | Can be safely copied or duplicated byte-for-byte using `std::memcpy` or `std::memmove`. |
| **POD (Plain Old Data)**| `is_pod<T>` | Combination of Trivial and Standard-Layout (deprecated in C++20 in favor of granular traits). |
| **Literal Types** | `is_literal_type<T>` | Types that can be instantiated, manipulated, and evaluated inside `constexpr` compile-time functions. |

---

## 5. Type Modifiers, Qualifiers, and Deduction

### 5.1 CV-Qualifiers (`const` and `volatile`)
Any base type $T$ can be decorated with cv-qualifiers:

* **`const`**: Declares an object immutably bound after initialization. Modifying a `const` object results in compile errors (or undefined behavior via unsafe casting).
* **`volatile`**: Signals to the compiler that a value can be mutated by hardware, signals, or external threads outside normal program control flow. Suppresses read/write caching optimizations.
* **`const volatile`**: Used for read-only hardware registers that change unpredictably outside software execution.

### 5.2 Storage Duration & Linkage Keywords
Specifies the memory lifecycle and visibility scope of objects:

* **`static`**: Gives local variables persistent lifetime across function calls, or limits global symbols to internal linkage.
* **`extern`**: Specifies external linkage, referencing a variable defined in another compilation unit.
* **`thread_local`**: Allocates distinct instance variables per execution thread (C++11).
* **`mutable`**: Allows a class data member to be modified even inside `const` member functions.

### 5.3 Type Deduction Syntax

Modern C++ provides type deduction mechanisms to simplify code and improve generic programming capabilities:

```cpp
const int target = 42;

// auto: Deduces type via value decay (strips top-level const and references)
auto a = target;             // Type deduced as: int

// auto&: Retains const and reference characteristics
auto& b = target;            // Type deduced as: const int&

// decltype(expr): Queries exact type and value category of an expression
decltype(target) c = 10;     // Type deduced as: const int

// decltype(auto): Deduces exact type while preserving references and qualifiers
decltype(auto) getRef() {
    return target;           // Returns: const int
}
```

---

## 6. Summary: Property Comparison Matrix

| Type Category | Dynamic Allocation? | Safe for `std::memcpy`? | Can possess member functions? | Implicit conversions? |
| :--- | :--- | :--- | :--- | :--- |
| **Fundamental (int, float)** | No | Yes | No | Yes (Integral/Promotions) |
| **Unscoped Enum** | No | Yes | No | Yes (Implicit to `int`) |
| **Scoped Enum (`enum class`)**| No | Yes | No | No (Requires explicit cast) |
| **Trivial Struct / Class** | Optional | Yes | Yes | Explicit / User-defined |
| **Non-Trivial Class** | Optional | **No** (Triggers UB) | Yes | Explicit / User-defined |
| **Reference (`T&`, `T&&`)** | No (Alias) | **No** | No | Refers to underlying target |
