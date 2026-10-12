# C++ Advanced Skill

An expert-level AI skill for modern C++ engineering (C++20/23, gated C++26). Version 3.2.0. `SKILL.md` is the router: output mode, which reference to open, and the five triggers (English and Portuguese). Feature rules live in `references/`, one home each.

## Features

- **Core Architecture & Language:**
  - Modern language standards and clean code principles.
  - Advanced memory management and resource safety.
  - Efficient data manipulation and sequences, including the C++23 range-adaptor family (`zip`, `enumerate`, `adjacent`, `chunk`, `ranges::to`).
  - `std::optional`/`std::expected` monadic minimization (`and_then`, `transform`, `or_else`, `value_or`, `emplace`, `reset`, `swap`).
  - `std::bit_cast` type-punning with explicit argument/aliasing constraints.
- **C++26 Frontier Features (gated behind confirmed toolchain support):**
  - Static reflection (`^^`, `[: :]`, `std::meta::*`) for eliminating boilerplate.
  - Contracts (`pre`/`post`/`contract_assert`) as executable documentation, layered over — never replacing — mandatory guard clauses.
  - `std::execution` senders/receivers for structured-concurrency async pipelines.
  - `std::hive` for stable-pointer object pooling and `std::inplace_vector` for fixed-capacity inline storage.
  - Trivial relocation, `std::optional<T&>`, `std::indirect` / `std::polymorphic`, `std::function_ref`, `std::copyable_function`.
  - Pack indexing, pack-introducing structured bindings, `template for`, and `#embed`.
  - `<linalg>` and `std::submdspan` over `std::mdspan`, plus `<simd>` and saturating arithmetic.
  - `std::hazard_pointer` and `std::rcu` for lock-free reclamation. Erroneous behavior is not a license to read uninitialized memory.
- **Modern Design Patterns:**
  - Robust object creation and lifecycle management.
  - High-performance structural composition.
  - Type-safe behavioral abstractions and logic.
- **Performance, Security & Tooling:**
  - Data-oriented design and hardware-aware optimization.
  - Low-level security, anti-reversing, and defensive engineering.
  - Safe concurrency and thread synchronization.
  - Professional build systems and security hardening.
  - Stable interoperability and exception safety boundaries.
  - Advanced memory pattern analysis and scanning.
- **Quality Audit Protocols (report-only, no code output):**
  - Memory Leak Hunter — leaked allocations, OS handles, `shared_ptr` cycles, dangling references, with severity rubric and ASan/CRT-debug-heap/Valgrind corroboration.
  - Race Condition Hunter — unsynchronized shared state, deadlock ordering, atomics/memory-ordering misuse, thread-lifetime races, with ThreadSanitizer corroboration and `std::execution` structural fixes.
  - Over-Engineering Auditor — needless polymorphism, indirection, reinvention, and configurability, reported as terse cut/replace findings.

## Auto-Invocation

This skill is **model-invoked**. The `description` lists the branches (write/review, refactor plan, the three audits, C++26 vocabulary). Portuguese aliases (`analisar`, `plano de refatoração`, `auditoria de memória`, `auditoria de corrida`, `simplificar`) are in `SKILL.md`. Anti-RE and signature scanning load only when the user asks for them.

## Usage

This skill is designed to be used with AI agents and LLM interfaces to assist in architectural reviews, implementation of complex C++ systems, and targeted audits (`"memory audit"`, `"race audit"`, `"over-engineering audit"`) of existing C++ codebases. Trigger phrases:

- `"refactor plan"` / `"analisar"` — architectural review + tiered modernization roadmap.
- `"implement phase [X]"` / `"implementar correção [N]"` — generate code for a selected roadmap item or audit finding.
- `"memory audit"` / `"auditoria de memória"` — Memory Leak Hunter report.
- `"race audit"` / `"auditoria de corrida"` — Race Condition Hunter report.
- `"over-engineering audit"` / `"simplificar"` — Over-Engineering Auditor report.

## C++26 Gating Policy

C++26 is gated. The rule and the per-feature macros live in `SKILL.md` and the matching reference file. The skill does not emit ungated C++26.
