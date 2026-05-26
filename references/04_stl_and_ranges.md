# STL, Ranges, Views & Advanced Algorithms

This document dictates the strict use of the Standard Template Library (STL) in modern C++ (C++20/C++23). The primary mandate is: **Declarative programming over imperative execution.** The AI must leverage lazy evaluation, compile-time checks, and range-based composability to eliminate stateful, manual loops.

## 1. The "No Raw Loops" Mandate
* **Algorithmic Default:** Never write a manual `for` or `while` loop if a standard algorithm can express the same intent. Manual loops are error-prone and hide the actual logic.
* **`std::ranges` over `std::`:** You must exclusively use the constrained algorithms in the `std::ranges::` namespace (e.g., `std::ranges::find_if`, `std::ranges::transform`). Never use the legacy iterator-pair algorithms (`std::begin`, `std::end`) unless interfacing with legacy APIs.

## 2. Views & Lazy Evaluation (`std::views`)
* **Zero-Allocation Pipelines:** Always use `std::views` (like `filter`, `transform`, `take`, `reverse`) to manipulate data sequences. Views do not allocate memory and evaluate lazily. Do not allocate intermediate `std::vector` containers just to transform data.
* **Composability (The Pipe Operator):** Chain view operations using the pipe operator (`|`). This ensures the data flow is read linearly from left to right.
  `auto result = data | std::views::filter(is_valid) | std::views::transform(extract_value);`
* **Immutability:** Always bind the result of a view pipeline to a `const auto` variable to guarantee the view itself is not accidentally mutated.

## 3. Projections & Callables
* **Range Projections:** C++20 ranges support projections. You must use them to simplify algorithms instead of writing verbose lambdas. 
  * *Bad:* `std::ranges::sort(users, [](const User& a, const User& b) { return a.id < b.id; });`
  * *Good:* `std::ranges::sort(users, std::less<>{}, &User::id);`
* **Lambdas:** When a custom callable is strictly necessary, use concise lambdas. Prefer explicit captures (`[this]`, `[&data]`) over implicit catch-alls (`[&]`, `[=]`). Keep lambda bodies extremely short; if they exceed 3 lines, extract them into a private `constexpr` method.

## 4. Compile-Time Algorithms
* **`constexpr` STL:** In C++20 and above, most STL algorithms and containers (`std::vector`, `std::string`) are `constexpr` friendly. If the input data is known at compile-time, you must wrap the algorithmic pipeline in a `constexpr` or `consteval` function to execute the logic entirely during compilation.

---

## Code Generation Directives for AI Models

1. **No Implementation Comments:** `.cpp` files or inline lambda bodies must contain strictly zero comments. The intent must be self-evident through the declarative nature of the view pipelines, descriptive variables, and projections.
2. **English Exclusivity:** All code constructs, variables, and literal strings must be natively written in English.
3. **Purity:** Methods that process ranges but do not modify internal state must be explicitly marked `const`. Return values from all range pipelines must be rigidly checked using `[[nodiscard]]`.q