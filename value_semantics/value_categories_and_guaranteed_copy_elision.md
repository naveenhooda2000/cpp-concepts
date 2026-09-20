# C++ Value Categories and Guaranteed Copy Elision

## 1. Taxonomy of Value Categories

In C++11, the value category system was completely redesigned to support **move semantics**. Every expression in C++ is classified based on two independent properties:

1. **Has Identity?** Does the expression refer to an object with a persistent memory location that can be identified and accessed elsewhere?
2. **Can Be Moved From?** Can resources be safely stolen from the expression because its lifetime is ending or it has been explicitly marked as expiring?

---

### Value Category Hierarchy

```
                  Expression
                 /          \
            glvalue        rvalue
           /       \      /      \
      lvalue        xvalue        prvalue
```

---

### Primary Value Categories

| Category | Identity? | Movable? | Description & Key Examples |
| :--- | :---: | :---: | :--- |
| **lvalue** | **Yes** | **No** | Identifies a persistent object or function. Addressable via `&`. <br>• Named variables (`int x`) <br>• Lvalue references (`T&`) <br>• Functions returning lvalue references (`std::vector::operator[]`) <br>• String literals (`"hello"`) |
| **prvalue** (*pure rvalue*) | **No** | **Yes** | Computes a value or initializes an object. Represents temporary data without persistent identity. <br>• Non-string literals (`42`, `true`) <br>• Arithmetic expressions (`a + b`) <br>• Function calls returning non-reference types (`std::string("hi")`) <br>• Lambda expressions |
| **xvalue** (*expiring value*) | **Yes** | **Yes** | Identifies an object whose lifetime is about to end or has explicitly been marked as eligible for move semantics. <br>• The result of `std::move(x)` <br>• Functions returning rvalue references (`T&&`) <br>• Subobject access on an xvalue (`std::move(pair).first`) |

---

### Composite Value Categories

* **glvalue** (*generalized lvalue*) = `lvalue` $\cup$ `xvalue`  
  *Expressions that have **identity**.*
* **rvalue** = `prvalue` $\cup$ `xvalue`  
  *Expressions that **can be moved from**.*

---

## 2. Why C++11 Split Rvalues into prvalues and xvalues

Prior to C++11, C++ had only **lvalues** and **rvalues**. An rvalue simply meant "anything that wasn't an lvalue" (temporary values).

When **move semantics** and **rvalue references (`T&&`)** were introduced in C++11 to avoid expensive deep copies, the language needed to distinguish between two fundamentally different types of movable expressions:

1. **Pure Temporaries (`prvalue`):** Expressions created on the fly with no name and no lifetime beyond the current expression (e.g., `a + b`).
2. **Expiring Named Objects (`xvalue`):** Objects that *do* have names and persistent locations in memory, but where the developer explicitly states: *"I am finished with this object, treat it as temporary and steal its resources."*

---

### The Role of `std::move`

`std::move` does not perform any runtime moves. It is merely a compile-time cast:

```
std::move(x) implies static_cast}<T&&>(x)
```

* `x` is an **lvalue** (has identity, cannot be safely moved without permission).
* `std::move(x)` is an **xvalue** (retains identity, but signals that its resources can be stolen).

Without **xvalues**, C++ would have no category to represent an existing named object that temporarily acts like a movable rvalue.

---

## 3. C++17 Guaranteed Copy Elision and prvalues

In C++11 and C++14, a `prvalue` was defined as an actual temporary object created in memory. Compilers were permitted to perform Copy/Move Elision (RVO/NRVO) as an optimization, but:
* The class still required an accessible copy or move constructor.
* Compilers were not strictly obligated to elide copies in every scenario.

---

### The C++17 Redefinition of prvalues

C++17 redefined a **prvalue** from "a temporary object" to an **initialization recipe or blueprint**.

* **Deferred Construction:** A `prvalue` expression does not create a temporary object on its own.
* **Direct Target Initialization:** When a `prvalue` initializes an object, construction happens directly in the target memory location.
* **Temporary Materialization:** A temporary object is instantiated *only* when a `prvalue` must be converted to a `glvalue` (e.g., binding to a `const T&`).

---

### Factory Functions for Non-Copyable / Non-Movable Types

Because prvalue copy elision is **guaranteed by the C++17 standard**, types without copy or move constructors can now be returned by value:

```cpp
#include <iostream>

struct NonMovable {
    NonMovable() = default;
    NonMovable(const NonMovable&) = delete; // No copy constructor
    NonMovable(NonMovable&&) = delete;      // No move constructor
};

NonMovable create_object() {
    return NonMovable(); // prvalue (initialization recipe)
}

int main() {
    // C++11/14: Compilation Error (Move/Copy constructor required even if elided)
    // C++17+:   Valid Code (Guaranteed zero copies, zero moves)
    NonMovable obj = create_object(); 
}
```

---

### Execution Steps in C++17

1. `NonMovable()` evaluates as a **prvalue** (a recipe for constructing `NonMovable`).
2. `return NonMovable()` passes the recipe out of the function.
3. `NonMovable obj = create_object();` uses the recipe to construct `obj` directly in `main`'s stack frame.
4. Zero temporaries are created, zero move/copy constructors are called.
5. 
