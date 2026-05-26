# Behavioral Design Patterns & Functional Paradigms

This document dictates the architectural rules for implementing Behavioral Patterns in modern C++ (C++20/C++23). The core philosophy is to modernize GoF (Gang of Four) patterns by replacing heavy object-oriented inheritance with functional paradigms, value semantics, and modern type-safe unions.

## General Behavioral Directives

1. **Prefer Value Semantics and Callables:** Before creating an entire class hierarchy for a behavioral pattern (like Command or Strategy), evaluate if a simple `std::function` or a lambda can achieve the same result with less boilerplate.
2. **Compile-Time Behavior:** Whenever the behavior is known at compile-time, prefer Policy-Based Design (templated strategies) over dynamic polymorphism (virtual tables).
3. **Strict State Management:** When patterns require state transfer or observation, you must guarantee thread safety and prevent memory leaks (dangling pointers or cyclic references).

---

## 1. Chain of Responsibility

Passes requests along a chain of handlers.

* **Architectural Rule:** Each handler must explicitly decide whether to process the request, pass it forward, or reject it.
* **Modern C++ Implementation:**
  * Manage the chain lifecycle using `std::unique_ptr<Handler>` for the `nextHandler`.
  * The `handle()` method must return `[[nodiscard]] std::optional<Result>` or `std::expected<Result, Error>`. If a handler cannot process the request, it returns the result of `nextHandler->handle()`. If the chain ends without resolution, return `std::nullopt` or an error.

## 2. Command

Encapsulates a request as an object, allowing for parameterization, queuing, and undo operations.

* **Architectural Rule:** Differentiate between simple execution and undoable state mutations.
* **Modern C++ Implementation:**
  * **Simple Commands:** Use `std::function<void()>` instead of a full class hierarchy.
  * **Undoable Commands:** Implement an interface with `virtual void execute() = 0;` and `virtual void undo() = 0;`.
  * Store commands in a `std::vector<std::unique_ptr<Command>>` to implement history/undo stacks.

## 3. Iterator

Traverses a collection without exposing its underlying representation.

* **Architectural Rule:** The classic GoF Iterator pattern is **deprecated** in this skill. 
* **Modern C++ Implementation:**
  * **Strict Mandate:** You must use C++20 `std::ranges` and `std::views`. Do not implement custom `.hasNext()` or `.next()` methods.
  * If wrapping a custom data structure, expose standard C++ iterators (`begin()`, `end()`) so the structure instantly integrates with the `<ranges>` library.

## 4. Mediator

Reduces chaotic dependencies between objects by forcing them to collaborate via a mediator.

* **Architectural Rule:** Use to decouple complex UI components or micro-architectures.
* **Modern C++ Implementation:**
  * Components must not hold strong references (`std::shared_ptr`) to the Mediator if the Mediator holds strong references to the components (cyclic leak). 
  * Pass the Mediator to components via constructor injection as a raw reference `Mediator&` (if lifetimes are strictly bound) or use an event bus architecture with `std::function` callbacks.

## 5. Memento

Saves and restores the previous state of an object without revealing its implementation details.

* **Architectural Rule:** The Memento object must be strictly immutable.
* **Modern C++ Implementation:**
  * The `createMemento()` method must be `[[nodiscard]]` and return by value (e.g., a `struct` containing the state).
  * Do not allocate Mementos on the heap unless absolutely necessary; favor stack allocation and value semantics.

## 6. Observer (Publish-Subscribe)

Defines a subscription mechanism to notify multiple objects about events.

* **Architectural Rule:** The Observer pattern is the #1 cause of memory leaks in C++ due to cyclic references. You must enforce strict lifecycle management.
* **Modern C++ Implementation:**
  * The Subject (Publisher) must store Observers as a collection of weak pointers: `std::vector<std::weak_ptr<Observer>>`.
  * Before notifying, the Subject must lock the weak pointer: `if (auto spt = weakObs.lock()) { spt->update(); }`. Clean up dead pointers from the vector periodically.

## 7. State

Allows an object to alter its behavior when its internal state changes.

* **Architectural Rule:** Avoid allocating new state objects on the heap for every state transition.
* **Modern C++ Implementation:**
  * **Dynamic State:** If states are complex, hold `std::unique_ptr<State>` and use `std::exchange` or `std::move` during transitions.
  * **Zero-Allocation FSM (Preferred):** For strictly defined state machines, use `std::variant<StateA, StateB>` inside the Context class and handle state-dependent behavior using `std::visit`. This completely eliminates heap allocation and virtual dispatches.

## 8. Strategy

Defines a family of algorithms, encapsulates each one, and makes them interchangeable.

* **Architectural Rule:** Decouple the algorithm logic from the host context.
* **Modern C++ Implementation:**
  * **Runtime Strategy:** Inject the behavior via constructor as a `std::function<Signature>` or `std::unique_ptr<Strategy>`.
  * **Compile-Time Strategy (Policy):** If the strategy doesn't change at runtime, inject it as a template parameter `<typename StrategyPolicy>`.

## 9. Template Method

Defines the skeleton of an algorithm in the base class, letting subclasses override specific steps.

* **Architectural Rule:** Strictly apply the **Non-Virtual Interface (NVI) idiom**.
* **Modern C++ Implementation:**
  * The public interface method (the skeleton) must be non-virtual.
  * The customizable steps must be `protected` or `private` pure virtual methods: `virtual void step() = 0;`.
  * This guarantees the base class retains control over the execution flow and pre/post-conditions.

## 10. Visitor

Separates algorithms from the objects on which they operate.

* **Architectural Rule:** The classic OOP Visitor (using `accept` and `visit` double-dispatch) is **deprecated** in modern C++.
* **Modern C++ Implementation (Pattern Matching):**
  * Represent the hierarchy of element types as a `std::variant<TypeA, TypeB, TypeC>`.
  * Implement the visitor as a struct with overloaded `operator()` for each type.
  * Apply the visitor using `std::visit(Visitor{}, variantElement)`. This provides compile-time exhaustiveness checking and better cache locality.

---

## Code Generation Directives for AI Models

1. **No Implementation Comments:** `.cpp` files must contain zero comments. The intent must be self-evident through the use of standard modern paradigms (e.g., `std::visit`, `std::weak_ptr::lock`).
2. **English Exclusivity:** All code constructs must be in English.
3. **Purity:** Methods that do not modify state must be explicitly marked `const`. Check return values rigorously using `[[nodiscard]]`.