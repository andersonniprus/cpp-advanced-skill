# Performance, Data-Oriented Design & Micro-optimizations

This document dictates the architectural rules for writing mechanically sympathetic, high-performance C++ code. The AI must prioritize CPU cache locality, branch prediction, and contiguous memory access over abstract object-oriented modeling.

## 1. Data-Oriented Design (DOD)
* **Struct of Arrays (SoA) over Array of Structs (AoS):** When processing large volumes of data linearly, do not use collections of large objects (`std::vector<ComplexObject>`). Instead, split the data into parallel arrays (`std::vector<int> ids`, `std::vector<float> positions`) to maximize CPU cache line utilization and enable automatic SIMD vectorization by the compiler.
* **Cache-line Alignment:** Align frequently accessed structures to the cache line boundary (typically 64 bytes) using `alignas(std::hardware_destructive_interference_size)` to prevent false sharing across multiple threads.

## 2. Branching & Predictability
* **Branchless Programming:** Avoid conditional branching (`if`/`else`) inside hot loops. Prefer mathematical computations, bitwise operations, or conditional moves (cmov) to keep the CPU pipeline full.
* **Likely/Unlikely Attributes:** In C++20, if a branch is heavily skewed (e.g., an error check that rarely fails), you must explicitly hint the compiler using `[[likely]]` and `[[unlikely]]`.
  `if (!result) [[unlikely]] { return std::unexpected(Error); }`

## 3. Allocation & Hot Paths
* **Heap Allocation Avoidance:** Inside hot loops or rendering frames, heap allocations (`new`, `std::make_unique`, pushing to `std::vector` without `reserve`) are strictly forbidden. Pre-allocate all required memory during the initialization phase (Memory Pooling/Arenas) and reuse it.
* **Pass by Value for Small Trivially Copyables:** Do not pass small, trivially copyable types (like `std::span`, `std::string_view`, or structs under 16 bytes) by `const&`. Pass them by value to avoid the indirection overhead of a pointer read.

## 4. Numerics on `mdspan` (`<linalg>`, C++26)
* **BLAS-Like Operations Over Hand-Rolled Loops:** For matrix/vector-heavy code (transforms, physics, ML inference glue) built on `std::mdspan` (C++23), use the C++26 `<linalg>` algorithms (e.g. `std::linalg::matrix_product`, `add`, `scale`) instead of hand-written nested loops — they map to vendor-tuned BLAS backends where available and are exception- and bounds-contract-consistent with the rest of the standard library.
* **Gating:** `<linalg>` is a C++26 addition layered on `std::mdspan`; confirm toolchain support before emitting it, and fall back to a manually written, unit-tested loop over `std::mdspan` otherwise.

---

## Code Generation Directives for AI Models

1. **No Implementation Comments:** `.cpp` files must contain zero comments. Cache alignment and branchless logic must be evident through clean structure and descriptive variable names.
2. **English Exclusivity:** All code constructs, variables, and literal strings must be natively written in English.
3. **Purity:** Methods must avoid side effects. Use `[[nodiscard]]` aggressively to ensure computations are consumed.