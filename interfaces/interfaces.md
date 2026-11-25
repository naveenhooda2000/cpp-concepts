# Implementing and Overriding Interfaces in Standard C++

While C++ does not provide an explicit `interface` keyword, the Standard Library exposes several polymorphic base classes, CRTP mixins, and trait-based customization points designed to be extended or overridden in user code.

This guide covers:
1. **Polymorphic Base Classes (Runtime Virtual Interfaces)**
2. **CRTP Mixin Base Classes (Compile-time Interfaces)**
3. **Template Customization Points & Ad-Hoc Interfaces**

---

## Table of Contents
- [1. std::exception](#1-stdexception)
- [2. std::pmr::memory_resource](#2-stdpmrmemory_resource)
- [3. std::streambuf](#3-stdstreambuf)
- [4. std::locale::facet (e.g., std::numpunct)](#4-stdlocalefacet-eg-stdnumpunct)
- [5. std::ranges::view_interface](#5-stdrangesview_interface)
- [6. std::enable_shared_from_this](#6-stdenable_shared_from_this)
- [7. std::hash](#7-stdhash)
- [8. Custom Allocator (std::allocator_traits)](#8-custom-allocator-stdallocator_traits)

---

## 1. `std::exception`

### When to Use
Implement or inherit from `std::exception` (or standard subclasses like `std::runtime_error`) whenever you need domain-specific exceptions in a subsystem. This allows catching components higher up the stack to intercept your custom errors polymorphicly alongside standard errors using `catch (const std::exception& e)`.

### How to Implement
- Derive from `std::exception`.
- Provide a `noexcept` virtual destructor (implicitly generated if defaulted).
- Override `const char* what() const noexcept`.

### Implementation Example

```cpp
#include <exception>
#include <string>
#include <iostream>

class DatabaseConnectionException : public std::exception {
public:
    DatabaseConnectionException(std::string host, int port, std::string detail)
        : m_message("DB Connection to " + host + ":" + std::to_string(port) + 
                    " failed: " + detail) {}

    // Required override
    const char* what() const noexcept override {
        return m_message.c_str();
    }

private:
    std::string m_message;
};

// Usage
void connect(const std::string& host, int port) {
    throw DatabaseConnectionException(host, port, "Connection timed out (ETIMEDOUT)");
}
```

---

## 2. `std::pmr::memory_resource`

### When to Use
Introduced in C++17, Polymorphic Memory Resources (`std::pmr`) allow you to pass custom allocators (e.g., arenas, stack buffers, fixed-size block pools) to standard containers (`std::pmr::vector`, `std::pmr::string`, etc.) **without changing the container's compile-time type**.

### How to Implement
Derive from `std::pmr::memory_resource` and override its three protected pure virtual methods:
1. `do_allocate(std::size_t bytes, std::size_t alignment)`: Allocates raw memory with the requested byte count and alignment.
2. `do_deallocate(void* p, std::size_t bytes, std::size_t alignment)`: Reclaims the memory.
3. `do_is_equal(const std::pmr::memory_resource& other) const noexcept`: Checks identity or equality against another memory resource instance.

### Implementation Example (Linear Monotonic Arena)

```cpp
#include <memory_resource>
#include <vector>
#include <cstddef>
#include <new>
#include <iostream>

class BumpArenaResource : public std::pmr::memory_resource {
public:
    explicit BumpArenaResource(std::size_t capacity)
        : m_capacity(capacity),
          m_buffer(static_cast<std::byte*>(::operator new(capacity))),
          m_offset(0) {}

    ~BumpArenaResource() override {
        ::operator delete(m_buffer);
    }

    BumpArenaResource(const BumpArenaResource&) = delete;
    BumpArenaResource& operator=(const BumpArenaResource&) = delete;

protected:
    void* do_allocate(std::size_t bytes, std::size_t alignment) override {
        // Calculate aligned address
        std::size_t space = m_capacity - m_offset;
        void* current_ptr = m_buffer + m_offset;
        
        void* aligned_ptr = std::align(alignment, bytes, current_ptr, space);
        if (!aligned_ptr) {
            throw std::bad_alloc();
        }

        m_offset = (static_cast<std::byte*>(aligned_ptr) - m_buffer) + bytes;
        return aligned_ptr;
    }

    void do_deallocate(void* /*p*/, std::size_t /*bytes*/, std::size_t /*alignment*/) override {
        // Monotonic arena: individual deallocations are no-ops.
        // Memory is reset or freed collectively in bulk.
    }

    bool do_is_equal(const std::pmr::memory_resource& other) const noexcept override {
        return this == &other;
    }

private:
    std::size_t m_capacity;
    std::byte*  m_buffer;
    std::size_t m_offset;
};

// Usage
void runPmrExample() {
    BumpArenaResource arena(1024 * 1024); // 1MB arena
    
    // Type is simply std::pmr::vector<int>
    std::pmr::vector<int> vec(&arena);
    for (int i = 0; i < 1000; ++i) {
        vec.push_back(i);
    }
}
```

---

## 3. `std::streambuf`

### When to Use
Implement a custom stream buffer whenever you need to route stream I/O (`std::ostream`, `std::istream`, or `std::iostream`) into a custom destination or from an unconventional source (e.g., in-memory ring buffers, logging systems, network sockets, or GUI text widgets).

### How to Implement
- Derive from `std::streambuf` (or `std::basic_streambuf<CharT>`).
- For **output streams**, override:
  - `overflow(int_type c)`: Called when the buffer is full or for unbuffered characters.
  - `sync()`: Called on flush (`std::endl`, `std::flush`).
  - Optionally `xsputn(const char_type* s, std::streamsize count)` for optimized bulk writes.
- For **input streams**, override:
  - `underflow()`: Reads the next character without advancing the get pointer.
  - `uflow()`: Reads the next character and consumes it.

### Implementation Example (Redirecting `std::ostream` to a Custom Logger)

```cpp
#include <streambuf>
#include <ostream>
#include <string>
#include <iostream>

class LoggerStreamBuf : public std::streambuf {
public:
    explicit LoggerStreamBuf(std::string prefix) 
        : m_prefix(std::move(prefix)) {}

protected:
    // Handles single character insertion when unbuffered
    int_type overflow(int_type ch) override {
        if (ch != traits_type::eof()) {
            if (ch == '\n') {
                flushBuffer();
            } else {
                m_buffer += traits_type::to_char_type(ch);
            }
        }
        return ch;
    }

    // Handles stream flushes
    int sync() override {
        flushBuffer();
        return 0;
    }

private:
    void flushBuffer() {
        if (!m_buffer.empty()) {
            std::cout << "[" << m_prefix << "] " << m_buffer << "\n";
            m_buffer.clear();
        }
    }

    std::string m_prefix;
    std::string m_buffer;
};

// Usage
void runStreamExample() {
    LoggerStreamBuf buf("ENGINE");
    std::ostream logStream(&buf);

    logStream << "Physics engine initialized." << std::endl;
    logStream << "Frame rate: " << 120 << " FPS" << std::endl;
}
```

---

## 4. `std::locale::facet` (e.g., `std::numpunct`)

### When to Use
Use custom facets when you need to change localization rules, such as custom number formatting, date delimiters, grouping separators, or boolean string representation (`"True"`/`"False"` instead of `"true"`/`"false"`).

### How to Implement
- Inherit from one of the standard facet classes (e.g., `std::numpunct<char>`).
- Override protected virtual methods prefixing with `do_` (e.g., `do_thousands_sep()`, `do_grouping()`, `do_decimal_point()`).
- Imbue the custom locale into the target stream.

### Implementation Example (Custom Comma Grouping & Decimal Separator)

```cpp
#include <locale>
#include <iostream>

class EuropeanNumberPunct : public std::numpunct<char> {
protected:
    // Custom decimal point: comma instead of period
    char do_decimal_point() const override {
        return ',';
    }

    // Custom thousands separator: dot instead of comma
    char do_thousands_sep() const override {
        return '.';
    }

    // Grouping by sets of 3 digits
    std::string do_grouping() const override {
        return "\3";
    }
};

// Usage
void runLocaleExample() {
    // std::locale manages facet lifetime via ref-counting
    std::locale customLoc(std::cout.getloc(), new EuropeanNumberPunct());
    std::cout.imbue(customLoc);

    std::cout << 1234567.89 << '\n'; // Outputs: 1.234.567,89
}
```

---

## 5. `std::ranges::view_interface`

### When to Use
Introduced in C++20, `std::ranges::view_interface` uses the Curiously Recurring Template Pattern (CRTP) to synthesize a rich, standard-compliant Range/View API for your custom collections.

### How to Implement
- Inherit from `std::ranges::view_interface<Derived>`.
- The derived class must minimally provide `begin()` and `end()`.
- Based on the iterator capabilities returned by `begin()`, `view_interface` automatically generates `empty()`, `operator bool()`, `data()`, `size()`, `front()`, `back()`, and `operator[]`.

### Implementation Example (Custom Sliding Window View)

```cpp
#include <ranges>
#include <vector>
#include <iostream>
#include <span>

template <typename T>
class ChunkPairView : public std::ranges::view_interface<ChunkPairView<T>> {
public:
    ChunkPairView(const T* data, std::size_t size)
        : m_data(data), m_size(size) {}

    struct Iterator {
        using iterator_category = std::forward_iterator_tag;
        using value_type        = std::pair<T, T>;
        using difference_type   = std::ptrdiff_t;
        using pointer           = const value_type*;
        using reference         = value_type;

        const T* ptr;

        reference operator*() const { return { *ptr, *(ptr + 1) }; }
        Iterator& operator++() { ptr += 2; return *this; }
        bool operator==(const Iterator& other) const { return ptr >= other.ptr; }
    };

    Iterator begin() const { return Iterator{ m_data }; }
    Iterator end() const { 
        return Iterator{ m_data + (m_size - (m_size % 2)) }; 
    }

private:
    const T* m_data;
    std::size_t m_size;
};

// Usage
void runViewExample() {
    std::vector<int> nums = {1, 2, 3, 4, 5, 6};
    ChunkPairView<int> pairView(nums.data(), nums.size());

    // Automatically acquired methods via view_interface
    if (!pairView.empty()) {
        auto first_pair = pairView.front();
        std::cout << "(" << first_pair.first << ", " << first_pair.second << ")\n";
    }
}
```

---

## 6. `std::enable_shared_from_this`

### When to Use
Derive from `std::enable_shared_from_this<T>` when an object managed by a `std::shared_ptr` needs to safely produce a new `std::shared_ptr<T>` instance from inside its own member methods (for example, when registering itself into event listeners or asynchronous callback chains).

*Note: Calling `shared_from_this()` on an object not already owned by a `std::shared_ptr` throws `std::bad_weak_ptr`.*

### Implementation Example

```cpp
#include <memory>
#include <vector>
#include <iostream>

class EventDispatcher;

class NetworkSession : public std::enable_shared_from_this<NetworkSession> {
public:
    void registerListener(EventDispatcher& dispatcher);
    void onMessage(const std::string& msg) {
        std::cout << "Handling message: " << msg << '\n';
    }
};

class EventDispatcher {
public:
    void subscribe(std::shared_ptr<NetworkSession> session) {
        m_subscribers.push_back(session);
    }
private:
    std::vector<std::shared_ptr<NetworkSession>> m_subscribers;
};

void NetworkSession::registerListener(EventDispatcher& dispatcher) {
    // Safely obtains a shared_ptr pointing to 'this'
    dispatcher.subscribe(shared_from_this());
}
```

---

## 7. `std::hash`

### When to Use
Specialize `std::hash<Key>` in namespace `std` to allow custom types or structs to serve as keys in standard hash-based containers (`std::unordered_map`, `std::unordered_set`).

### How to Implement
- Specialize `struct std::hash<CustomType>`.
- Implement `std::size_t operator()(const CustomType& key) const noexcept`.
- Combine the hashes of individual fields (typically via bit shifting and the Golden Ratio prime `0x9e3779b9`).
- Ensure `operator==` is defined for your type.

### Implementation Example

```cpp
#include <string>
#include <functional>
#include <unordered_set>

struct UserId {
    std::string domain;
    uint64_t id;

    bool operator==(const UserId& other) const noexcept {
        return id == other.id && domain == other.domain;
    }
};

// Custom hash specialization
template <>
struct std::hash<UserId> {
    std::size_t operator()(const UserId& u) const noexcept {
        std::size_t h1 = std::hash<std::string>{}(u.domain);
        std::size_t h2 = std::hash<uint64_t>{}(u.id);

        // Standard hash-combine formula
        return h1 ^ (h2 + 0x9e3779b97f4a7c15ULL + (h1 << 6) + (h1 >> 2));
    }
};

// Usage
void runHashExample() {
    std::unordered_set<UserId> activeUsers;
    activeUsers.insert({"internal", 42});
}
```

---

## 8. Custom Allocator (`std::allocator_traits`)

### When to Use
Implement a custom C++ allocator when containers need custom allocation schemes (e.g., shared memory, fixed-size static pools) while remaining compatible with classic allocator-aware containers (`std::vector<T, Alloc>`, `std::list<T, Alloc>`).

### How to Implement
Since C++11, `std::allocator_traits` provides sensible defaults for boilerplate requirements (`rebind`, `construct`, `destroy`, `pointer`, etc.). A minimal custom allocator requires:
1. `using value_type = T;`
2. `T* allocate(std::size_t n)`
3. `void deallocate(T* p, std::size_t n) noexcept`
4. Copy constructor and equality comparison operators (`operator==`, `operator!=`).

### Implementation Example (Minimal Malloc-Based Allocator)

```cpp
#include <cstddef>
#include <cstdlib>
#include <new>
#include <vector>
#include <iostream>

template <typename T>
class CustomMempoolAllocator {
public:
    using value_type = T;

    CustomMempoolAllocator() noexcept = default;

    template <typename U>
    CustomMempoolAllocator(const CustomMempoolAllocator<U>&) noexcept {}

    [[nodiscard]] T* allocate(std::size_t n) {
        if (n > std::size_t(-1) / sizeof(T)) {
            throw std::bad_alloc();
        }
        
        void* p = std::malloc(n * sizeof(T));
        if (!p) {
            throw std::bad_alloc();
        }
        return static_cast<T*>(p);
    }

    void deallocate(T* p, std::size_t /*n*/) noexcept {
        std::free(p);
    }

    template <typename U>
    bool operator==(const CustomMempoolAllocator<U>&) const noexcept {
        return true;
    }

    template <typename U>
    bool operator!=(const CustomMempoolAllocator<U>&) const noexcept {
        return false;
    }
};

// Usage
void runAllocatorExample() {
    std::vector<int, CustomMempoolAllocator<int>> customVector;
    customVector.push_back(10);
    customVector.push_back(20);
}
```
