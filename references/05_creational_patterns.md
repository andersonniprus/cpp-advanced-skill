# Creational Design Patterns & Object Lifecycle

This document provides advanced theoretical foundations and strict implementation directives for Creational Design Patterns in modern C++. The core philosophy is to decouple object creation from object usage while guaranteeing absolute memory safety, enforcing the Open/Closed Principle, and utilizing modern C++ compiler features.

## General Creational Directives

1. **Strict Ownership Transfer:** Creational patterns exist to create and return resources. Every factory method, builder termination, or clone operation must return ownership via `std::unique_ptr<T>`. Never return raw pointers (`T*`).
2. **Mandatory `[[nodiscard]]`:** The act of creating an object is an expensive allocation and resource acquisition. Every method responsible for returning a newly created object must be marked `[[nodiscard]]`.
3. **No Legacy Allocation:** The keywords `new` and `delete` are strictly banned. Use `std::make_unique` or `std::make_shared` exclusively.

---

## 1. Factory Method

Provides an interface for creating objects in a superclass, allowing subclasses to alter the type of objects that will be created.

* **Architectural Rule:** Use the Factory Method to encapsulate the instantiation logic of concrete products that implement a common interface.
* **Modern C++ Implementation:**
  * The factory method must be declared as `[[nodiscard]] virtual std::unique_ptr<Product> createProduct() const = 0;`.
  * Ensure the base `Product` class has a `virtual ~Product() = default;` to prevent undefined behavior upon destruction.
  * Concrete factories must override the method and return `std::make_unique<ConcreteProduct>()`.
* **Constraint:** Do not use parameters to decide which subclass to instantiate (anti-pattern) unless implementing a parameterized factory/registry. True Factory Method relies on polymorphism.

## 2. Abstract Factory

Produces families of related objects without specifying their concrete classes.

* **Architectural Rule:** Use this pattern when your system must support multiple families of products (e.g., cross-platform UI elements) and you want to ensure that products from one family are not accidentally mixed with another.
* **Modern C++ Implementation:**
  * Group related `[[nodiscard]] virtual std::unique_ptr<ProductX> createX() const = 0;` methods within the Abstract Factory interface.
  * Inject the Abstract Factory into the client using constructor injection via reference (`const AbstractFactory&`) or `std::shared_ptr<const AbstractFactory>` if the client needs to extend the factory's lifecycle.

## 3. Builder

Constructs complex objects step by step. Allows producing different types and representations of an object using the same construction code.

* **Architectural Rule:** Apply the Builder pattern to solve the "Telescoping Constructor Anti-Pattern" (classes with dozens of constructor parameters).
* **Modern C++ Implementation:**
  * **Fluent Interfaces & Deducing `this`:** In C++23, implement fluent interfaces using explicit object parameters (deducing `this`) to cleanly return references to the builder without virtual table overhead or CRTP boilerplate.
  * If limited to C++20, return `Builder&` from configuration steps.
  * **The Build Step:** The final method (e.g., `build()`) must be `[[nodiscard]]`. It should transfer ownership of the internally built object to the caller using `std::move`.
  * **Destructive Build:** Once `build()` is called, the Builder should be left in a valid but empty state.

## 4. Prototype

Copies existing objects without making the code dependent on their classes.

* **Architectural Rule:** Use Prototype when object creation is heavily dependent on runtime state, making standard instantiation via factories too costly or complex.
* **Modern C++ Implementation:**
  * Standard C++ covariant return types do not work natively with `std::unique_ptr`. 
  * **The Clone Idiom:** Declare a public interface: `[[nodiscard]] virtual std::unique_ptr<Base> clone() const = 0;`.
  * Concrete classes implement it by utilizing a private or protected copy constructor: `return std::make_unique<Concrete>(*this);`.
  * Ensure the base class has a virtual destructor. Delete the assignment operator if polymorphic copying is the sole intention.

## 5. Singleton (Naïve vs. Thread-safe)

Ensures that a class has only one instance, while providing a global access point.

* **Architectural Rule:** Singleton is widely considered an anti-pattern because it introduces global state and hides dependencies. Prefer Dependency Injection. Use Singleton *only* for strictly hardware-bound resources (e.g., an underlying GPU context).
* **Modern C++ Implementation (Meyers' Singleton):**
  * If you must implement it, use the **Meyers' Singleton** approach. It is inherently thread-safe in C++11 and onwards.
  * Define a static local variable inside the access method:
    `static Singleton& getInstance() { static Singleton instance; return instance; }`
  * **Strict Deletions:** You must explicitly delete the copy constructor, move constructor, copy assignment operator, and move assignment operator to prevent accidental duplication.
    `Singleton(const Singleton&) = delete;`
    `Singleton& operator=(const Singleton&) = delete;`
    `Singleton(Singleton&&) = delete;`
    `Singleton& operator=(Singleton&&) = delete;`

---

## Code Generation Directives for AI Models

1. **Header Files (`.hpp`):** May include a single `@brief` above the class declaration specifying the pattern being used.
2. **Implementation Files (`.cpp`):** Must contain exactly **zero comments**. The architecture, method chaining, variable names, and explicit use of `std::move` and `std::make_unique` must be entirely self-documenting.
3. **Immutability:** Mark all factory creation methods as `const` unless the factory maintains internal caching or object pooling state.