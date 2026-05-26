# Memory Scanning, SIMD & Signature Parsing

This document dictates the architectural mandates for implementing high-performance memory pattern scanners (Aho-Corasick, Boyer-Moore, or brute-force vectorized). The AI must prioritize hardware-level parallelism (SIMD), compile-time evaluation, and safe memory probing.

## 1. Compile-Time Signature Parsing
* **No Runtime String Parsing:** Parsing IDA-style byte signatures (e.g., `"48 8B 05 ? ? ? ?"`) at runtime wastes CPU cycles. 
* **`consteval` Translation:** You must use `consteval` functions or variadic template metaprogramming to parse string signatures into raw byte arrays and wildcard mask arrays entirely during compilation.
* **Data-Oriented Patterns:** The resulting compile-time signature must be structured to maximize cache locality for the scanning algorithm, separating the bytes to match from the mask boolean/bitfield.

## 2. SIMD Acceleration (Vectorization)
* **AVX2 / AVX-512 Mandate:** For large memory region scanning, naive byte-by-byte looping is strictly forbidden. You must utilize intrinsic SIMD instructions (`<immintrin.h>`).
* **First-Byte Matching:** Use `_mm256_set1_epi8` to broadcast the first byte of the signature, and `_mm256_cmpeq_epi8` coupled with `_mm256_movemask_epi8` to scan 32 bytes simultaneously. Only perform the full signature mask check when the first byte matches.

## 3. Safe Memory Probing
* **Page Protection Checks:** Before scanning an arbitrary memory region, you must query the page protections (using `VirtualQueryEx` or `VirtualQuery`).
* **Avoid `PAGE_NOACCESS`:** Strictly skip pages marked as `PAGE_NOACCESS` or `PAGE_GUARD`. Attempting to vectorize a read into a guard page will cause an immediate Access Violation (Exception 0xC0000005).

## 4. Hardware Sympathy
* **Unrolling & Prefetching:** Loop unrolling should be left to the compiler via optimization flags (`/O2`), but explicit hardware prefetching (`_mm_prefetch`) can be used if profiling indicates memory bandwidth is the bottleneck for gigabyte-scale scans.
* **Alignment:** When loading data into SIMD registers from external memory, assume the memory is unaligned. Always use unaligned load intrinsics (e.g., `_mm256_loadu_si256`) to prevent hardware faults.

---

## Code Generation Directives for AI Models

1. **No Implementation Comments:** `.cpp` files must contain zero comments. SIMD intrinsics are notoriously complex; their intent must be clarified by wrapping them in highly descriptive, well-named inline functional wrappers.
2. **English Exclusivity:** All code constructs, variables, and literal strings must be natively written in English.
3. **Purity:** Scanning functions must never modify the memory they are inspecting. They must take `const std::span<const std::uint8_t>` or raw `const std::uint8_t*` and explicitly mark the method as `const`.