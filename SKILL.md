---
name: cpp-advanced-skill
description: "Advanced C++20/23/26 skill. Use when writing, reviewing, or modernizing C++ headers and sources; for a refactor plan or analyse/analise; for a memory audit (leaks, dangling, use-after-free), race audit (data races, deadlocks), or over-engineering/YAGNI/simplify audit; or when the user mentions std::optional/std::expected monads, std::ranges, std::bit_cast, RAII, design patterns, Windows anti-RE, or C++26 (reflection, contracts, std::execution, std::hive, std::inplace_vector, std::simd, trivial relocation, hazard_pointer, rcu, #embed)."
metadata:
  version: "3.2.0"
  scope: "provider-neutral-agent-harness"
  file_policy: "markdown-only"
---

# Advanced C++ Architect

Generate modern C++ (C++20/23, gated C++26) with strict ownership, standard-library-first design, and mechanical sympathy. For existing code, run evidence-only audits. Do not invent APIs, feature-test macro names, or compiler support.

## Output mode

Pick one mode from the user request. Do not mix them.

| Request | Output |
|---|---|
| Code only ("write", "generate", "implement") | Separate `.hpp` and `.cpp` fences. Bold filename before each fence (`**Engine.hpp**`). No preamble. No comments in `.cpp`. English identifiers. `const` and `[[nodiscard]]` where they apply. |
| Explanation ("why", "how does", "explain") | Short technical answer. Code only if it proves a point. |
| Plan or audit (triggers below) | Markdown report. No implementation until the user picks an item. |

Header documentation (Doxygen `@brief` in `.hpp`) is allowed. Implementation comments are not, including in code-only mode.

## C++26 gating

C++26 features in this skill are gated. Before emitting any of them, confirm the toolchain or wrap the use in the feature-test macro documented in that feature's reference, with a C++23 fallback. Never emit ungated C++26. If the macro name is uncertain for that compiler, ask instead of guessing. The authoritative feature list and fallbacks live in the reference files, not here.

## What to read

Open only the rows that match the request. Do not load the other reference files.

| Task | Read |
|---|---|
| Generate or review ordinary C++ | [01](references/01_core_guidelines.md), [02](references/02_modern_features.md), [03](references/03_memory_management.md), [04](references/04_stl_and_ranges.md), [14](references/14_optional_and_monadic_ops.md) |
| Refactor plan | [01](references/01_core_guidelines.md) plus the file for each topic you actually flag |
| Memory audit | [16](references/16_memory_leak_audit.md), then [03](references/03_memory_management.md) only to name a fix |
| Race audit | [17](references/17_race_condition_audit.md), then [09](references/09_concurrency_and_threading.md) only to name a fix |
| Over-engineering audit | [18](references/18_over_engineering_audit.md) |
| Reflection, contracts, or senders | [15](references/15_reflection_cpp26.md), [19](references/19_contracts_cpp26.md), or [20](references/20_senders_receivers_cpp26.md) |
| Hot path, numerics, SIMD | [10](references/10_performance_and_dod.md) |
| Build, FFI | [11](references/11_build_and_tooling.md), [12](references/12_interop_and_ffi.md) |
| Anti-RE, syscall, or signature scan — only if the user asked | [08](references/08_security_and_production.md), [13](references/13_memory_scanning.md) |
| A named GoF pattern | [05](references/05_creational_patterns.md), [06](references/06_structural_patterns.md), or [07](references/07_behavioral_patterns.md) |

Default shape when no pattern was requested: return `unique_ptr` from factories, use `std::variant` instead of a virtual hierarchy with one implementation, and use Pimpl only for a real ABI boundary. Container and callable choice is [03](references/03_memory_management.md) §9 and [02](references/02_modern_features.md) §6.

## Triggers

English or Portuguese. On match, stop default code generation and follow the protocol.

### 1. Refactor plan — "refactor plan", "analyse", "analyze", "analise", "analisar", "plano de refatoração"

No implementation. Architectural review, then:

1. Flag concrete replacements: monadic `optional`/`expected` for `has_value()` chains; `bit_cast` for punning; ranges for index loops; and, only if gated support is confirmed, `hive`, `inplace_vector`, contracts, `std::execution`, reflection, `function_ref`, trivial relocation, `<simd>`, hazard pointers, or `rcu`.
2. Roadmap: **Phase 1** syntax, `[[nodiscard]]`, `const`, `<format>`, monadic chains, pack indexing. **Phase 2** smart pointers, `pmr`, ranges, `bit_cast`, `jthread`, `inplace_vector`, `hive`, contracts. **Phase 3** cache layout, `<simd>`/`<linalg>`, reflection, `std::execution`, hazard pointers/`rcu`.
3. Give a technical alternative for each major shift.
4. End with: *Which Phase or specific point would you like me to implement first?*

### 2. Implement — "implement phase [X]", "implement fix [N]", "implementar fase", "implementar correção"

Generate only that item, in code-only output mode. Keep the project compiling before a further refactor.

### 3. Memory audit — "memory audit", "find memory leaks", "auditoria de memória", "vazamento de memória"

Follow [16](references/16_memory_leak_audit.md). Severity with file and line. Recommend ASan, the MSVC debug heap, or Valgrind. End by asking which finding to fix.

### 4. Race audit — "race audit", "find race conditions", "thread safety audit", "auditoria de corrida", "condição de corrida"

Follow [17](references/17_race_condition_audit.md). Severity with file and line. Recommend ThreadSanitizer. If `std::execution` is confirmed, a lifetime race may be fixed by a sender pipeline ([20](references/20_senders_receivers_cpp26.md)). End by asking which finding to fix.

### 5. Over-engineering audit — "over-engineering audit", "simplify", "yagni audit", "simplificar", "auditoria de complexidade"

Follow [18](references/18_over_engineering_audit.md). One line per finding: `Location — what to cut — what replaces it`. Skip empty categories. End by asking which finding to cut.
