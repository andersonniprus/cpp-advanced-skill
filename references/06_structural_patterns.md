# Structural Design Patterns & Composition

This document defines the strict architectural rules for implementing Structural Patterns in modern C++ (C++20/C++23). The primary mandate for any AI agent generating code in this domain is: **Favor Object Composition over Class Inheritance.**

## General Structural Directives

1. **Zero-Cost Abstractions:** When creating Facades or Proxies over contiguous memory or strings, use `std::span<T>` and `std::string_view` to provide structural views without allocating or copying underlying data.
2. **Strict Composition:** When a pattern requires one object to "wrap" or "contain" another (e.g., Decorator, Adapter), the wrapper must hold the inner object via `std::unique_ptr<T>` (for exclusive ownership) or a reference `T&` (if the wrapper's lifetime is strictly scoped within the wrapped object's lifetime).
3. **Compile-Time vs. Runtime:** Evaluate if the structural relationship can be resolved at compile-time. Use templates and CRTP (Curiously Recurring Template Pattern) combined with Deducing `this` for static polymorphism when dynamic runtime switching is not explicitly required.

---

## 1. Adapter

Allows objects with incompatible interfaces to collaborate.

* **Architectural Rule:** Strictly separate the target interface from the adaptee.
* **Modern C++ Implementation:**
  * **Object Adapter (Preferred):** The adapter implements the target interface and holds a `std::unique_ptr<Adaptee>` (or `Adaptee&` via constructor injection).
  * **Class Adapter (Restricted):** Use private multiple inheritance *only* if the Adapter must override protected behavior of the Adaptee. Otherwise, prefer the Object Adapter pattern.
  * All adapted methods must be marked `[[nodiscard]]` if they return values or state codes.

## 2. Bridge (The Pimpl Idiom)

Splits a large class into two separate hierarchies—abstraction and implementation.

* **Architectural Rule:** In C++, the Bridge pattern is fundamentally realized via the **Pimpl (Pointer to Implementation) idiom** to achieve ABI stability and drastically reduce compilation times.
* **Modern C++ Implementation:**
  * In the `.hpp` file, declare a forward-declared `struct Impl;` and hold it via `std::unique_ptr<Impl> pImpl;`.
  * The destructor `~Abstraction()` must be declared in the header but defined in the `.cpp` file to allow `std::unique_ptr` to evaluate the size of `Impl`.
  * Use `const std::unique_ptr<Impl>` if the implementation details are immutable after construction.

## 3. Composite

Composes objects into tree structures to represent part-whole hierarchies.

* **Architectural Rule:** Treat individual objects and compositions of objects uniformly.
* **Modern C++ Implementation:**
  * The parent `Composite` class must manage its children using a container of smart pointers: `std::vector<std::unique_ptr<Component>>`.
  * **Iteration:** When executing operations across the tree, do not use raw `for` loops. Expose a view or utilize `std::ranges::for_each` to traverse the children.
  * Prevent cyclic references. If a child needs a reference to its parent, it must use a raw non-owning pointer (`Component*`) or `std::weak_ptr<Component>` (if the tree uses `shared_ptr`).

## 4. Decorator

Attaches new behaviors to objects dynamically by placing them inside special wrapper objects.

* **Architectural Rule:** Use to prevent "class explosion" caused by subclassing every possible combination of behaviors.
* **Modern C++ Implementation:**
  * The Decorator base class must take ownership or reference of the component: `std::unique_ptr<Component> wrappedComponent;`.
  * Constructor injection mandates moving the component into the decorator: `Decorator(std::unique_ptr<Component> c) : wrappedComponent(std::move(c)) {}`.
  * Ensure methods pass through arguments efficiently using perfect forwarding `std::forward<Args>(args)...` if wrapping complex variadic templates.

## 5. Facade

Provides a simplified interface to a library, framework, or complex set of classes.

* **Architectural Rule:** The Facade isolates the client from the subsystem's complexity. It must not leak subsystem headers into the client's namespace.
* **Modern C++ Implementation:**
  * The Facade class should encapsulate the initialization and orchestration of multiple sub-components.
  * If the subsystem initialization can fail, the Facade's creation method or initialization call must return a `[[nodiscard]] std::expected<SuccessType, ErrorEnum>` to enforce error handling by the client.

## 6. Flyweight

Fits more objects into RAM by sharing common parts of state (intrinsic state).

* **Architectural Rule:** Strictly divide state into intrinsic (shared, immutable) and extrinsic (contextual, mutable).
* **Modern C++ Implementation:**
  * The Flyweight Factory must manage a cache of shared states (e.g., using `std::unordered_map`).
  * The factory must return `std::shared_ptr<const Flyweight>` to guarantee that the intrinsic state remains immutable and is automatically cleaned up when no longer referenced by any context.

## 7. Proxy

Provides a placeholder for another object to control access, delay initialization, or handle caching.

* **Architectural Rule:** The Proxy and the Real Subject must share the same interface.
* **Modern C++ Implementation:**
  * **Virtual Proxy (Lazy Initialization):** Hold a `std::unique_ptr<RealSubject>`. Instantiate it via `std::make_unique` only upon the first method call.
  * **Protection Proxy:** Perform security/authorization checks and return early if the check fails before delegating to the Real Subject.

---

## Code Generation Directives for AI Models

1. **Rule of Zero / Five:** Any structural class managing resources directly (like Pimpl) must explicitly define or delete the copy/move constructors and assignment operators to prevent shallow copy bugs.
2. **Comments Directive:** The `.cpp` files must contain zero comments. The AI must rely on descriptive naming, early returns, and modern syntax (like `auto const&`) to make the structural relationships obvious.
3. **Language:** All variable names, class names, strings, and structures must be natively written in English.