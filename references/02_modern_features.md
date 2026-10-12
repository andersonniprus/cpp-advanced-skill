# Modern C++ Features & Compile-Time Evaluation

This document dictates the strict use of modern C++ (C++20/C++23) language features. The AI must prioritize compile-time evaluation, robust type safety, and zero-overhead abstractions. Legacy C++ paradigms (macros, raw templates without concepts, SFINAE) are strictly prohibited.

## 1. Compile-Time Evaluation (The `const` Family)
Maximize the compiler's capability to enforce correctness and execute logic at compile-time to reduce runtime overhead to absolute zero.
* **`const`:** Default state for any variable that does not mutate after initialization.
* **`constexpr`:** Use for variables, functions, and constructors that *can* and *should* be evaluated at compile-time. Prefer `constexpr` algorithms over runtime execution whenever the input domain is known statically.
* **`consteval` (C++20):** Mandatory for functions that *must* execute at compile-time (immediate functions). Use this for compile-time string hashing, static parsing, or metaprogramming generation.
* **`constinit` (C++20):** Use for static or thread-local variables to guarantee static initialization and completely eliminate the Static Initialization Order Fiasco.

## 2. Type Deduction & Auto
Follow a disciplined "Almost Always Auto" (AAA) approach, prioritizing readability without sacrificing strict memory semantics.
* **`auto`:** Use to prevent repeating lengthy type names, especially for iterators, ranges, and smart pointers (e.g., `auto ptr = std::make_unique<ComplexType>();`).
* **Strict Qualifiers:** Naked `auto` creates copies. You must explicitly qualify references to reflect your intent:
  * `const auto&` for read-only observation.
  * `auto&` for mutable observation.
  * `auto&&` for forwarding references in template contexts or range-based loops over proxy objects.

## 3. Metaprogramming & Constraints
* **Concepts (C++20):** Mandatory for all template constraints. Replace generic `typename T` with explicit Concepts (e.g., `std::integral auto`, `std::ranges::range auto`) to provide immediate, readable compiler errors. Do not use legacy `std::enable_if` (SFINAE) under any circumstances.
* **`if constexpr` (C++17):** Use for static branching inside templates. Discard invalid branches at compile-time to simplify logic and avoid compilation errors on unused branches.
* **Pack Indexing (C++26):** Use `pack...[index]` to access a single element of a template or function parameter pack directly (`Ts...[0]`, `args...[N]`). This replaces recursive template metaprogramming or `std::index_sequence`/`std::tuple_element` gymnastics that existed solely to extract the Nth pack element. Gate behind confirmed C++26 toolchain support; fall back to `std::tuple_element_t<N, std::tuple<Ts...>>` otherwise.
* **Structured bindings that introduce a pack (C++26):** A binding may end in `...` so the tail of a tuple-like object becomes a pack (`auto [head, tail...] = tuple;`). Use this instead of `std::apply` plus an index sequence when only the tail is forwarded. Same gating rule as pack indexing.
* **`#embed` (C++26):** Embed a binary file as a compile-time integer sequence (`#embed "table.bin"`) instead of a Python/xxd-generated `unsigned char` array. Gate it; fall back to a `constexpr std::array<std::byte, N>` produced by the build. Do not parse the same bytes from disk at startup when the file is a build input.

## 4. Object Orientation & "Deducing this" (C++23)
* **Explicit Object Parameters:** In C++23, utilize Deducing `this` to dramatically simplify class design and template hierarchies.
* **Eliminate Overloads:** Replace duplicated `const` and non-`const` member functions with a single template method taking `this` explicitly:
  `template<typename Self> auto&& getValue(this Self&& self) { return std::forward<Self>(self).value; }`
* **CRTP Simplification:** Use Deducing `this` to implement the Curiously Recurring Template Pattern (CRTP) without needing the base class to be a template of the derived class, removing massive amounts of boilerplate.

## 5. Type Punning & Casting
* **`std::bit_cast` (C++20):** When converting between different types of the same size (type punning) where both are trivially copyable, you must use `std::bit_cast<DestTy>(source)`. It is safe, `constexpr` friendly, and avoids Undefined Behavior inherent to `reinterpret_cast`.
* **Argument Contract:** `std::bit_cast` requires `sizeof(To) == sizeof(From)` and both `To` and `From` trivially copyable; the compiler enforces this via `static_assert`-equivalent constraints, so a failing call is a compile error, not latent UB. Unlike `reinterpret_cast`, `To` need not be default-constructible — the destination object is produced directly from the source bit pattern.
* **Canonical Uses:** Float/integer bit manipulation (`std::bit_cast<std::uint32_t>(floatValue)` instead of a `union` or `memcpy`), and network/binary-protocol field extraction from a `std::array<std::byte, N>` into a POD struct. Pair with `std::byteswap` (C++23) when the source bytes are a different endianness than the host: `std::bit_cast<std::uint32_t>(bytes)` then `std::byteswap(value)` if crossing endianness, never a hand-rolled shift-and-mask sequence.
* **What `bit_cast` Does Not License:** Producing a pointer/reference value via `bit_cast` and then dereferencing it as an unrelated type is still governed by strict aliasing — `bit_cast` legalizes the *value copy*, not arbitrary reinterpretation through the resulting pointer. Never use it to punch through a class hierarchy or reinterpret an opaque handle as a different pointer type; use it strictly for value-to-value bit reinterpretation between trivially copyable types.
* **Static Assertions:** In C++23, use `static_assert(false, "Error");` directly inside `if constexpr` blocks. Do not implement or use legacy `AlwaysFalse` template hacks.

## 6. Callables (owning vs non-owning)

Pick the cheapest type that matches the lifetime. Do not default to `std::function`.

| Need | Type |
|---|---|
| Non-owning, caller keeps the target alive for the call | `std::function_ref<Sig>` (C++26, `<functional>`). No allocation. |
| Owning, move-only (lambda with `unique_ptr`) | `std::move_only_function<Sig>` (C++23) |
| Owning, copyable, type-erased | `std::copyable_function<Sig>` (C++26). Prefer it over `std::function` when the toolchain has it |
| One concrete target, header can see the type | Template or `auto` parameter. No type erasure |

`function_ref` does not extend the target's lifetime. A `function_ref` stored past the callable it points at is a dangling call. Gate `function_ref` and `copyable_function`; fall back to a template parameter or `move_only_function` / `std::function`.

## 7. Formatting & I/O Strictness (The `<iostream>` Ban)
* **Strictly Forbid `<iostream>`:** The inclusion of `<iostream>` and the use of `std::cout`, `std::cin`, or `std::endl` are absolutely prohibited. They severely bloat the compiled binary size due to locale overhead and drastically degrade runtime execution performance.
* **Modern Formatting:** For string formatting and output, exclusively use C++20 `<format>` (e.g., `std::format`) or C++23 `<print>` (e.g., `std::println`).
* **Production Logging:** For runtime or diagnostic logging, never use standard streams. Rely on raw memory buffers, memory-mapped files, or specialized fast asynchronous loggers designed for low-latency environments.

---
