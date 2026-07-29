# C++ Advanced Skill

An expert-level AI skill for modern C++ engineering (C++20/23, gated C++26), specializing in high performance, security, low-level optimization, and evidence-only code-quality audits.

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
  - `std::hive` for stable-pointer object pooling.
  - Pack indexing (`pack...[N]`) replacing recursive pack-extraction metaprogramming.
  - `<linalg>` BLAS-like numerics over `std::mdspan`.
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

This skill is **model-invoked**: its `SKILL.md` `description` carries one trigger phrase per branch (write/review C++, refactor plan, memory audit, race audit, over-engineering audit, or a mention of monadic `optional`/`expected`, `std::ranges`, `std::bit_cast`, C++26 features, or Windows Anti-RE/ABI hardening) so a harness can fire it autonomously on any of those requests, without requiring the user to name the skill explicitly.

## Usage

This skill is designed to be used with AI agents and LLM interfaces to assist in architectural reviews, implementation of complex C++ systems, and targeted audits (`"memory audit"`, `"race audit"`, `"over-engineering audit"`) of existing C++ codebases. Trigger phrases:

- `"refactor plan"` / `"analyse <code>"` — architectural review + tiered modernization roadmap.
- `"implement phase [X]"` / `"implement fix [N]"` — generate code for a selected roadmap item or audit finding.
- `"memory audit"` / `"find memory leaks"` — Memory Leak Hunter report.
- `"race audit"` / `"find race conditions"` / `"thread safety audit"` — Race Condition Hunter report.
- `"over-engineering audit"` / `"simplify"` / `"yagni audit"` — Over-Engineering Auditor report.

## C++26 Gating Policy

Every C++26 feature (static reflection, contracts, `std::execution`, `std::hive`, pack indexing, `<linalg>`) is experimental and rolling out across toolchains. The skill never emits ungated C++26 code — each feature is checked against a `<version>` feature-test macro (or explicit user confirmation) with a fallback path before use.
