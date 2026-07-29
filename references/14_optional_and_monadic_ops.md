# `std::optional` / `std::expected` Monadic Minimization

This document dictates the strict use of the `std::optional<T>` (C++17) and `std::expected<T, E>` (C++23) monadic interface to eliminate manual `has_value()`/`if`-`else` boilerplate. The primary mandate is: **A hand-written branch that the standard library already expresses as a chained call is a defect, not a style choice.**

## 1. The Ban on Manual Presence Checks
* **Forbidden Pattern:** Never write `if (opt.has_value()) { use(opt.value()); } else { fallback(); }` when the equivalent monadic call exists. This pattern hides the data flow inside control flow and forces the reader to reconstruct the pipeline mentally.
* **Mandate:** Express presence/absence logic as a chain read top-to-bottom: acquire, transform, recover.

## 2. Mutating an Existing `optional` (No Manual Destroy-Reconstruct)
* **`emplace(args...)`:** Constructs the contained value in place, destroying any previously held value first. Never write `opt = std::nullopt; opt = T{args...};` — call `opt.emplace(args...)` directly; it is exception-safe and avoids the intermediate empty state.
* **`reset()`:** Destroys the contained value and returns the optional to the empty state. Never write `opt = std::optional<T>{};` or `opt = std::nullopt;` for this — `reset()` states the intent directly and is `noexcept`.
* **`swap(other)`:** Exchanges the states of two optionals (member `opt.swap(other)` or non-member `std::swap`). Never hand-roll a temporary-and-move dance to swap two optionals; the library implementation correctly handles the four presence-combination cases (both engaged, both empty, one engaged) without redundant construction.

## 3. Safe Extraction
* **`value_or(default_value)`:** The default choice for extracting a value with a fallback. Requires `T` to be copyable (or the fallback convertible to `T`). Prefer this over a ternary on `has_value()`.
* **`value()`:** Throws `std::bad_optional_access` if empty. Reserve for call sites where emptiness is already a proven-impossible invariant (documented in the calling context), not as a substitute for `value_or` or a monadic chain.
* **`operator*` / `operator->`:** Unchecked access — Undefined Behavior if empty. Only after an explicit `has_value()`/`if (opt)` guard in the same scope, typically inside performance-critical hot paths where the monadic wrappers' extra move/copy is profiled to matter.

## 4. Monadic Chaining (C++23)
* **`and_then(f)`:** Invoked only when engaged. `f` must return an `optional`-like type (`std::optional<U>` or `std::expected<U, E>`); the result is that returned value directly — no double-wrapping. Use for chaining fallible steps (`parse().and_then(validate).and_then(persist)`).
* **`transform(f)`:** Invoked only when engaged. `f` returns a plain `U`; the result is automatically wrapped (`std::optional<U>` / `std::expected<U, E>`). Use for infallible mappings (`opt.transform([](int v) { return v * 2; })`).
* **`or_else(f)`:** Invoked only when empty (`optional`) or holding an error (`expected`). `f` must return the same optional/expected type; use for recovery, lazy defaults, or logging-then-fallback. Never use `or_else` to perform a side effect on the success path — that is what `transform`/`and_then` are for.
* **Chain Direction:** Read a chain top-to-bottom as the happy path; `or_else` branches are the sole recovery point, kept at the end or interleaved only where recovery must happen before the next fallible step.

```cpp
auto result = parseRequest(raw)
    .and_then(validateSchema)
    .transform([](const Request& r) { return r.normalized(); })
    .or_else([](ParseError err) { return std::expected<Request, ParseError>{defaultRequest()}; });
```

## 5. `std::expected<T, E>` Parity and Divergence
* **Shared Surface:** `and_then`, `transform`, `or_else`, `value_or(default_value)` behave identically to `optional`, but operate over the error channel `E` instead of the empty state.
* **`transform_error(f)`:** `expected`-exclusive. Invoked only when holding an error; `f` maps `E` to a new error type `G`. Use to translate low-level errors (`std::errc`) into domain error enums as they cross architectural boundaries — never `catch`/rethrow for this.
* **`value()` / `error()`:** `value()` throws `std::bad_expected_access<E>` if the object holds an error; `error()` is UB if the object holds a value. Guard with `has_value()`/`operator bool` first, or prefer `value_or`/monadic chaining.
* **Return Discipline:** A function returning `std::expected<T, E>` must be constructed with `std::unexpected{errorValue}` on the failure path — never throw to signal the same failure the return type already models.

## 6. When *Not* to Chain
* If a step in the pipeline needs to branch on more than one independent condition (not just presence/absence or success/error), a monadic chain becomes harder to read than a guard-clause function. Extract the branch into a well-named `constexpr` helper instead of forcing a 4-way `and_then`/`or_else` chain.
* Hot loops calling `and_then`/`transform` with capturing lambdas can incur extra move/copy construction versus a manual branch; profile before rejecting the manual branch in that specific hot path (see [10_performance_and_dod.md](10_performance_and_dod.md)).

---

## Code Generation Directives for AI Models

1. **No Implementation Comments:** `.cpp` files must contain zero comments. Each stage of a monadic chain must be a descriptively named function or a short lambda whose intent is self-evident.
2. **English Exclusivity:** All code constructs, variables, and literal strings must be natively written in English.
3. **Purity:** Prefer free functions or `const` member functions as chain steps. Return values from every `optional`/`expected`-returning function must be rigorously checked using `[[nodiscard]]`.
