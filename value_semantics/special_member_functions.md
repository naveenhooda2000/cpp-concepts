# Comprehensive Guide: Implicit Special Member Function Generation in C++

## 1. The Big Six Special Member Functions

Modern C++ manages object lifecycles using six special member functions:

1. **Default Constructor:** `T()`
2. **Destructor:** `~T()`
3. **Copy Constructor:** `T(const T&)`
4. **Copy Assignment Operator:** `T& operator=(const T&)`
5. **Move Constructor:** `T(T&&)`
6. **Move Assignment Operator:** `T& operator=(T&&)`

---

## 2. Core Frameworks: Rule of 0 / 3 / 5

* **C++98 (Rule of 3):** If a class requires a custom **Destructor**, **Copy Constructor**, or **Copy Assignment**, it almost certainly requires all three to manage resources safely.
* **C++11 (Rule of 5):** The introduction of move semantics added **Move Constructor** and **Move Assignment**. Managing custom resources requires implementing or declaring all five.
* **Modern C++ (Rule of 0):** Classes should rely on modern RAII abstractions (`std::string`, `std::vector`, `std::unique_ptr`) to manage resources automatically. If a class declares zero special member functions, the compiler generates all six with default semantics.

---

## 3. Compiler Generation Pipeline

Compiler generation rules follow a clear hierarchy:

```
[ User declares ANY Constructor ] ────────► Suppresses Default Constructor
                                                  │
[ User declares ANY Copy/Destructor ] ────► Suppresses Automatic Moves
                                                  │
[ User declares ANY Move ] ───────────────► Suppresses Automatic Copies & Moves
```

### Detailed Generation Mechanics

1. **Default Constructor (`T()`):**
   * **Generated if:** No user-declared constructors of *any* kind exist.
   * **Suppressed if:** Any custom constructor (parameterized, copy, or move) is declared.

2. **Move Operations (`T(T&&)` and `operator=(T&&)`):**
   * **Generated if ALL of the following are true:**
     * No user-declared copy constructors.
     * No user-declared copy assignment operators.
     * No user-declared move operations.
     * No user-declared destructor.
   * **Key Concept:** Move operations are *all-or-nothing*. Declaring any custom copy, move, or destructor suppresses automatic generation of move operations.

3. **Copy Operations (`T(const T&)` and `operator=(const T&)`):**
   * **Generated if:** No user-declared move operations exist.
   * **Deleted if:** Any user-declared move constructor or move assignment operator exists.
   * **Deprecated behavior:** Declaring a custom destructor or copy operation deprecates (but still implicitly generates) the remaining copy operations for backward compatibility.

---

## 4. Generation Matrix

| User Declares... | Default Constructor | Destructor | Copy Constructor | Copy Assignment | Move Constructor | Move Assignment |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: |
| **Nothing (Rule of 0)** | **Generated** | **Generated** | **Generated** | **Generated** | **Generated** | **Generated** |
| **Any Custom Constructor** (e.g., `T(int)`) | <span style="color:red">**Disabled**</span> | **Generated** | **Generated** | **Generated** | **Generated** | **Generated** |
| **Destructor** | **Generated** | *User-defined* | **Generated\*** | **Generated\*** | <span style="color:red">**Disabled**</span> | <span style="color:red">**Disabled**</span> |
| **Copy Constructor** | <span style="color:red">**Disabled**</span> | **Generated** | *User-defined* | **Generated\*** | <span style="color:red">**Disabled**</span> | <span style="color:red">**Disabled**</span> |
| **Copy Assignment** | **Generated** | **Generated** | **Generated\*** | *User-defined* | <span style="color:red">**Disabled**</span> | <span style="color:red">**Disabled**</span> |
| **Move Constructor** | <span style="color:red">**Disabled**</span> | **Generated** | <span style="color:red">**Deleted**</span> | <span style="color:red">**Deleted**</span> | *User-defined* | <span style="color:red">**Disabled**</span> |
| **Move Assignment** | **Generated** | **Generated** | <span style="color:red">**Deleted**</span> | <span style="color:red">**Deleted**</span> | <span style="color:red">**Disabled**</span> | *User-defined* |

_\* Note: Implicit generation when a custom destructor or copy function is present is marked as deprecated in the standard._

---

## 5. Practical Guidelines

1. **Default to the Rule of Zero:** Rely on standard types to manage resource lifecycles.
2. **Apply the Rule of Five Explicitly:** If you declare any destructor, copy, or move function, explicitly declare all five using `= default` or `= delete`.

```cpp
class ExplicitResourceHandler {
public:
    ExplicitResourceHandler();
    ~ExplicitResourceHandler();

    // Explicitly preserve default move and copy behavior
    ExplicitResourceHandler(const ExplicitResourceHandler&) = default;
    ExplicitResourceHandler& operator=(const ExplicitResourceHandler&) = default;
    ExplicitResourceHandler(ExplicitResourceHandler&&) = default;
    ExplicitResourceHandler& operator=(ExplicitResourceHandler&&) = default;
};
```
