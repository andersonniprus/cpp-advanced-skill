# Over-Engineering & YAGNI Audit Protocol

This document defines the **Over-Engineering Auditor** persona: a review protocol that hunts for complexity the codebase pays for but does not use. It is the counterweight to every other reference file in this skill — modern idioms and design patterns are tools for a real problem, not default output. Adopt this protocol only under the "Over-Engineering Audit"/"Simplify" trigger; it produces a terse findings list, not code, and changes nothing on its own.

## Operating Principle
Every abstraction (interface, template parameter, indirection layer, design pattern) has exactly one job: make a *currently real* variability point cheap to change. An abstraction with no second implementation, no pending requirement, and no test exercising the variability is a cost with no return — flag it for deletion, not for "future-proofing" credit.

## 1. Needless Polymorphism
* **Interface With a Single Implementation:** A `virtual` interface or abstract base class with exactly one concrete subclass anywhere in the codebase, and no second implementation on a documented roadmap. Cut: inline the interface into the concrete class; drop the vtable indirection.
* **CRTP/Template Policy for Non-Performance Code:** A compile-time policy pattern used where runtime polymorphism would be equally fast and the code is not in a measured hot path. Cut: a plain class or `std::function` injection is easier to read and debug.
* **`std::variant`/`std::visit` for Two Trivially Distinguishable Cases:** A visitor-pattern-style variant dispatch where a single `if`/early-return on a bool or enum would read more directly. Reserve `variant`/`visit` for genuinely open, pattern-matched state machines ([07_behavioral_patterns.md](07_behavioral_patterns.md) §7/§10).

## 2. Needless Indirection
* **PIMPL Without an ABI-Stability Requirement:** The Pointer-to-Implementation idiom exists to decouple a library's binary interface from its internal layout. Applied to an internal, statically-linked, single-binary component with no external ABI consumer, it is pure compile-time and runtime overhead. Cut: expose the members directly (private, still header-hidden via access control, not via a second heap allocation).
* **Wrapper Classes Around a Single STL Type With No Added Invariant:** A class that only forwards every call to an internal `std::vector`/`std::string` member with no additional invariant, validation, or behavior. Cut: use the STL type directly, or a type alias if the name adds real domain meaning.
* **Factory/Builder for a Type With ≤3 Constructor Parameters and No Validation Step:** Builder pattern ([05_creational_patterns.md](05_creational_patterns.md) §3) solves the telescoping-constructor problem; applying it to a small aggregate is ceremony. Cut: a designated-initializer-friendly aggregate or a simple constructor.

## 3. Needless Reinvention
* **Custom Container/Allocator Duplicating an STL/`std::pmr` Type:** A hand-rolled ring buffer, small-vector, arena, or stable-pointer object pool that reimplements what `std::vector`, `std::deque`, `std::pmr::monotonic_buffer_resource`, `std::span`, or (C++26) `std::hive` ([03_memory_management.md](03_memory_management.md) §8) already provide, with no benchmark demonstrating the standard type is insufficient. Cut: use the standard type; if a benchmark does justify a custom type, keep the finding but downgrade it, and require the benchmark be attached.
* **Manual String/Byte Parsing Reimplementing `<charconv>`/`<format>`:** Hand-rolled integer/float parsing or formatting where `std::from_chars`/`std::to_chars`/`std::format` already exist and are faster and safer.
* **A Bespoke Event Bus/DI Container for a Single Producer and Single Consumer:** Infrastructure sized for N:M relationships applied to a fixed 1:1 relationship. Cut: a direct function call or a single `std::function` member.

## 4. Needless Configurability
* **Plugin/Strategy System With One Registered Strategy:** Runtime-selectable behavior (via `std::function`, registry map, or virtual strategy) where only one implementation has ever been registered and no second is planned. Cut: call the one implementation directly; reintroduce the seam when the second implementation actually arrives.
* **Compile-Time Feature Flags/Template Parameters Never Instantiated With a Second Value:** A template parameter or `#ifdef` axis with only one value used across the entire codebase. Cut: collapse to the single concrete value.
* **Configuration Options With No Reader:** A settings struct/JSON schema field that no code path ever branches on. Cut: remove the field and its plumbing.

## 5. Needless Premature Optimization
* **SIMD/Lock-Free/DOD Restructuring Without a Profile:** Hand-vectorized loops, custom lock-free structures, or SoA restructuring ([10_performance_and_dod.md](10_performance_and_dod.md)) applied to code with no profiler evidence it is a bottleneck. Cut: revert to the straightforward `std::ranges` pipeline or `std::mutex`-guarded structure; keep a note of where to re-apply the optimization if profiling later proves it necessary.
* **Hand-Written Memory Pool for Allocations Outside a Hot Loop:** `std::pmr` arenas exist to eliminate hot-loop allocation ([03_memory_management.md](03_memory_management.md) §7); applied to cold-path, infrequent allocations they add bookkeeping cost with no measurable benefit.

## 6. Report Format
One line per finding, terse: `Location — what to cut — what replaces it.` No prose paragraphs, no severity theater — over-engineering findings are binary (cut it or keep it), not graded. Group findings under the section headings above. Skip a section entirely if it has no findings; do not pad the report to look thorough.

---

## Report Generation Directives for AI Models

1. **Evidence-Only:** Every finding must name the exact file/class/function reviewed and confirm (by having actually checked) there is no second implementation, pending requirement, or profiling evidence before recommending a cut.
2. **English Exclusivity:** The report must be in English.
3. **No Rewrites Included:** This trigger produces findings only. Code changes happen only when the user selects a specific finding to implement (reuse Trigger 2's "Implement Fix" flow).
