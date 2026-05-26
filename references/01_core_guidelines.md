# C++ Core Guidelines & Architectural Purity

This document dictates the absolute baseline rules for physical code structure, clean code practices, and syntax purity. Every generated artifact must adhere to these structural constraints before any domain logic or design pattern is applied.

## 1. Physical Architecture & Separation
* **Strict Headers vs. Implementations:** Interfaces and class definitions belong exclusively in `.hpp` files. Implementations belong exclusively in `.cpp` files. Do not write inline method bodies inside the class declaration unless they are trivial `constexpr` getters or setters (max 1 line).
* **Header Guards:** Always use `#pragma once` at the very top of every header file.
* **Namespace Discipline:** * Never use `using namespace std;` (or any other namespace) in the global scope, and absolutely never inside a `.hpp` file.
  * Wrap your architecture in a descriptive, domain-specific namespace.

## 2. Documentation & Self-Documenting Code
* **Doxygen Headers Only:** The only permitted documentation is a highly professional, concise Doxygen-style `@brief` at the top of class declarations or complex method signatures within the `.hpp` file.
* **Implementation Blackout:** There must be absolutely ZERO comments in the `.cpp` file. If a block of code requires a comment to explain *what* it is doing, the code is fundamentally flawed. You must refactor it into smaller, well-named private methods.

## 3. Control Flow & Indentation
* **Early Returns (Guard Clauses):** Deeply nested `if-else` blocks are strictly forbidden. You must fail fast. Validate parameters and conditions at the very beginning of the function and `return` (or `throw`) immediately. Keep the "happy path" at the lowest possible indentation level.
* **Eliminate `else`:** If an `if` block ends with a `return` or `throw`, do not use an `else` block. Just continue the logic at the base scope.

## 4. Naming Conventions
* **Expressive English:** All names must be descriptive and in English. Avoid cryptic abbreviations. A variable should be named `employee_record` rather than `emp_rec` or `er`.
* **Standardization:**
  * Classes, Structs, and Concepts: `PascalCase`
  * Methods and Variables: `camelCase` or `snake_case` (but choose one and be strictly consistent).
  * Compile-time constants (`constexpr`): `kCamelCase` or `SCREAMING_SNAKE_CASE`.

## 5. Modern Error Handling
* **No Exceptions for Control Flow:** Do not use `try/catch` blocks for expected failure states (like reading an invalid memory address or failing to parse a file).
* **`std::expected` (C++23):** You must return `std::expected<T, ErrorEnum>` for functions that can fail. This strictly enforces the caller to handle the error branch.
* **`std::optional`:** Use `std::optional<T>` only when the absence of a value is a normal, expected state, and the specific reason for the absence (the error code) is completely irrelevant to the caller.

---

## Code Generation Directives for AI Models

1. **No Implementation Comments:** As stated above, `.cpp` files must contain zero comments. The intent must be self-evident through clean architecture, early returns, and precise naming.
2. **English Exclusivity:** All code constructs, variables, and literal strings must be natively written in English.
3. **Purity:** Methods that do not modify state must be explicitly marked `const`. Check return values rigorously using `[[nodiscard]]`. Never ignore an error state.