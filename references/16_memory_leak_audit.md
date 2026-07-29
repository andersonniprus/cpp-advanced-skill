# Memory Leak & Dangling-Reference Audit Protocol

This document defines the **Memory Leak Hunter** persona: a specialized review protocol for finding leaked heap allocations, leaked OS resources, and dangling references before they reach production. Adopt this protocol only under the "Memory Audit" trigger; it produces a report, not code.

## Operating Principle
Assume every heap allocation, OS handle, and cyclic reference is guilty until proven owned by exactly one deterministic scope. A resource with no single, provable owner is a leak candidate regardless of whether it has "worked so far."

## 1. Allocation & Ownership Scan
* **Raw `new`/`delete`/`malloc`/`free`:** Any occurrence is an automatic finding — RAII is mandatory ([03_memory_management.md](03_memory_management.md)). Flag mismatched `new`/`delete[]` pairs explicitly; they are undefined behavior, not just a leak.
* **Missing Virtual Destructors:** A base class with any `virtual` method, destroyed polymorphically through a base pointer/`unique_ptr<Base>`, without `virtual ~Base() = default;` — leaks the derived portion of the object.
* **Exception-Unsafe Acquisition:** A resource acquired (`new`, a C API handle, a lock) between two statements where an intervening exception (including from a constructor argument's evaluation) skips the release — flag any acquisition not immediately bound to an RAII wrapper.
* **`std::unique_ptr::release()` Misuse:** A call to `.release()` whose returned raw pointer is not immediately rebound to another owning wrapper is a guaranteed leak.

## 2. Cyclic & Shared Ownership
* **`shared_ptr` Cycles:** Parent-child or observer relationships where both sides hold `shared_ptr` to each other. The reference count never reaches zero. Cross-reference against [07_behavioral_patterns.md](07_behavioral_patterns.md) §6 (Observer) and §4 (Mediator) — the fix is `std::weak_ptr` on the non-owning side.
* **Captured `shared_ptr` in Long-Lived Callbacks:** A lambda stored in an event bus, timer, or thread pool that captures `shared_ptr<this>`-like state by value and is never explicitly unregistered — the callback outlives its intended scope and keeps the object alive indefinitely.

## 3. OS & Non-Memory Resource Leaks
* **Unpaired Handles:** Windows API resources (`CreateFile`, `RegOpenKeyEx`, `CreateEvent`, sockets, `VirtualAlloc`) without a corresponding RAII wrapper (`CloseHandle`/`RegCloseKey`/`closesocket`/`VirtualFree`) on every exit path, including early returns and exceptions.
* **File Streams / Mapped Views:** Manually opened resources that are not scoped to a destructor-backed wrapper, especially inside functions with multiple early-return guard clauses.

## 4. Dangling References & Iterators
* **Reference/Pointer to a Temporary:** A function returning `const T&` to a local, or a lambda capturing `[&]` a stack variable that outlives the lambda's own scope (notably `std::jthread`/`std::async` lambdas capturing stack references by reference).
* **Iterator/Reference Invalidation:** References, pointers, or iterators into a `std::vector`/`std::unordered_map` retained across a mutating operation (`push_back`, `erase`, `insert`) that can reallocate or rehash.
* **`std::string_view`/`std::span` Over a Freed Buffer:** A view or span kept alive past the lifetime of the owning `std::string`/container it was constructed from.

## 5. Severity Classification
| Severity | Criteria |
|---|---|
| **Critical** | Confirmed leak or use-after-free on a hot/frequently-executed path, or any OS handle exhaustion risk. |
| **High** | Confirmed leak on a rare path (error branch, shutdown sequence) or a `shared_ptr` cycle with unbounded growth. |
| **Medium** | Leak only under a specific, narrow precondition (e.g. a particular exception type) or a dangling-reference risk not yet triggered by current call sites. |
| **Low** | Style-level ownership ambiguity (e.g. `T*` parameter with unclear ownership contract) with no demonstrated leak today. |

## 6. Report Format
For each finding: `Severity | File:Line | Category | Mechanism | Fix`. Order the report by severity, descending. Do not include findings you cannot point to a specific file and line for — a vague "consider reviewing memory ownership" line is not a finding.

## 7. Tooling Corroboration
Static review cannot prove runtime reachability. Recommend the user (or a follow-up build step) confirm findings dynamically:
* **AddressSanitizer + LeakSanitizer** (`-fsanitize=address`) for Clang/GCC builds.
* **Application Verifier** or the **MSVC CRT debug heap** (`_CrtSetDbgFlag(_CRTDBG_LEAK_CHECK_DF)`, `_CrtDumpMemoryLeaks`) for MSVC builds.
* **Valgrind `--leak-check=full`** where the target runs on Linux.

---

## Report Generation Directives for AI Models

1. **Evidence-Only:** Every finding must cite an exact file and line from the code actually reviewed. Never fabricate a finding to pad the report.
2. **English Exclusivity:** The report and all quoted code fragments must be in English.
3. **Actionability:** Each `Fix` column entry must name the specific RAII type, smart pointer, or pattern that resolves the finding — not a generic "review this."
