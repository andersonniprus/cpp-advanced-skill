# Memory Management & Ownership Semantics

This document dictates the absolute rules for memory allocation, object lifecycles, and resource ownership in modern C++ (C++20/C++23). The primary mandate is: **Zero manual memory management and strict compilation-enforced ownership.**

## General Memory Directives

1. **The RAII Mandate:** Resource Acquisition Is Initialization (RAII) is non-negotiable. Every resource (heap memory, file handles, network sockets, thread locks) must be bound to a stack-allocated object's lifecycle. 
2. **Banned Keywords:** The explicit use of `new`, `delete`, `new[]`, `delete[]`, `malloc`, and `free` is strictly forbidden in generated code.
3. **Rule of Zero / Five:** Design classes so they do not require custom destructors, copy/move constructors, or assignment operators (Rule of Zero). If you must manage a low-level resource directly, you must explicitly define or delete all five (Rule of Five) to prevent shallow copies and double-free corruption.

---

## 1. Exclusive Ownership (`std::unique_ptr`)

Represents strict, sole ownership of a dynamically allocated resource.

* **Architectural Rule:** This must be your default choice for 95% of heap allocations. It has zero overhead compared to a raw pointer.
* **Modern C++ Implementation:**
  * **Creation:** Always use `std::make_unique<T>()`. This guarantees exception safety and prevents memory leaks if arguments throw during evaluation.
  * **Transfer of Ownership:** Use `std::move` to transfer ownership. A function accepting a `std::unique_ptr<T>` by value explicitly declares that it takes ownership of the resource.
  * **Return Types:** Factory methods and builders must always return `[[nodiscard]] std::unique_ptr<T>`. 

## 2. Shared Ownership (`std::shared_ptr`)

Represents a resource whose lifecycle is jointly managed by multiple owners via thread-safe atomic reference counting.

* **Architectural Rule:** `std::shared_ptr` is often used as a crutch by lazy designs. Use it **only** when the architecture explicitly demands shared, unpredictable lifecycles (e.g., asynchronous task queues, complex DAG graphs).
* **Modern C++ Implementation:**
  * **Creation:** Always use `std::make_shared<T>()`. This performs a single contiguous heap allocation for both the control block and the object, vastly improving CPU cache locality.
  * **Passing by Value vs. Const Reference:**
    * Pass by value `std::shared_ptr<T>` *only* if the function intends to extend the object's lifetime (e.g., storing it in a container or passing it to another thread).
    * Pass by `const std::shared_ptr<T>&` is an anti-pattern unless the function conditionally needs to take a copy. If the function just needs to *use* the object, pass a raw pointer `T*` or a reference `T&`.

## 3. Observation & Breaking Cycles (`std::weak_ptr`)

A non-owning observer of a resource managed by `std::shared_ptr`.

* **Architectural Rule:** Cyclic references between shared pointers cause memory leaks because the reference count never reaches zero. `std::weak_ptr` is the mandatory structural solution.
* **Modern C++ Implementation:**
  * Use for parent-to-child relationships (where the parent owns the child, but the child needs to signal the parent) and Observer patterns.
  * **Access:** You must check validity before use by converting it to a shared pointer via `.lock()` inside an `if` condition: 
    `if (auto spt = weakPtr.lock()) { spt->doWork(); }`

## 4. Non-Owning Observers (Raw Pointers & References)

Accessing data without dictating its lifecycle.

* **Architectural Rule:** Raw pointers (`T*`) are completely valid in modern C++, but strictly as non-owning, nullable observers. 
* **Modern C++ Implementation:**
  * If the function needs to observe an object but does not own it, and the object can logically be "null", pass it as a raw pointer `const T*`.
  * If the object cannot be null, pass it by reference `const T&`.
  * Never delete a raw pointer. Never assume a raw pointer conveys ownership.

## 5. Move Semantics & Forwarding

Optimizing resource transfers without expensive deep copies.

* **Architectural Rule:** Avoid unnecessary copies of large objects (like `std::vector`, `std::string`) by utilizing move semantics correctly.
* **Modern C++ Implementation:**
  * Use `std::exchange(member, newValue)` when implementing move assignment operators to simultaneously assign the new value and nullify the old state.
  * Use perfect forwarding (`std::forward<T>(arg)`) in variadic templates and factory wrappers to preserve the value category (lvalue/rvalue) of arguments.

## 6. Strong Typing for Memory Addresses
* **Avoid Raw Integrals for Addresses:** Do not pass raw `std::uintptr_t`, `std::uint64_t`, or `void*` throughout the codebase to represent memory locations.
* **Encapsulation:** Wrap memory addresses in strongly typed, `constexpr`-friendly wrapper classes (e.g., `Address`). Implement operator overloading (`+`, `-`, `<=>`) within the wrapper to guarantee safe bounds checking and valid arithmetic operations.
* **Contiguous Memory:** When handling buffers or arbitrary binary data, strictly use `std::span<const std::uint8_t>` instead of passing a raw pointer and a size parameter.

## 7. High-Performance Allocations & PMR
* **Avoid Heap Fragmentation:** Standard dynamic allocations (`new`, `make_unique`, or standard container growth) are strictly forbidden inside performance-critical hot loops.
* **Polymorphic Memory Resources (C++17):** For containers inside hot loops, you must utilize `std::pmr::vector`, `std::pmr::string`, or other `std::pmr` counterparts.
* **Stack-Backed Arenas:** You must bind these PMR containers to a `std::pmr::monotonic_buffer_resource` backed by a stack-allocated `std::array` buffer (or a pre-allocated heap arena). This guarantees mathematically zero heap allocations and absolute CPU cache locality during the loop's execution.

---

## Code Generation Directives for AI Models

1. **No Implementation Comments:** `.cpp` files must contain zero comments. The intent must be self-evident through the explicit use of `std::make_unique`, `std::move`, and RAII wrappers.
2. **English Exclusivity:** All code constructs, naming conventions, and logging must be in English.
3. **Purity:** Methods that do not modify state must be explicitly marked `const`. Functions that allocate and return memory must be rigorously checked using `[[nodiscard]]`.