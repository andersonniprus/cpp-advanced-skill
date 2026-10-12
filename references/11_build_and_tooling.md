# Build Systems, Modern CMake & Toolchain Hardening

This document dictates the architectural rules for generating build scripts and configuring compilers, specifically using CMake. The AI must prioritize a strict target-centric approach, reproducible builds, and aggressive compiler-level security hardening for the MSVC (Windows) environment.

## 1. Target-Centric Modern CMake (CMake 3.15+)
* **No Global State:** The use of legacy global directory-level commands is strictly forbidden. You must NEVER use `include_directories()`, `link_directories()`, `link_libraries()`, or `add_definitions()`.
* **The Target Paradigm:** Every configuration must be bound to a specific target using the `target_*` family of commands:
  * `target_include_directories(TargetName SCOPE dir)`
  * `target_link_libraries(TargetName SCOPE lib)`
  * `target_compile_definitions(TargetName SCOPE def)`
  * `target_compile_features(TargetName SCOPE cxx_std_23)`
* **Visibility Modifiers:** You must explicitly define the propagation scope using `PRIVATE`, `PUBLIC`, or `INTERFACE` for every `target_*` command. Never omit the scope modifier.

## 2. C++ Standard & Compiler Abstraction
* **Standard Enforcement:** Always enforce the C++ standard at the target level using `target_compile_features(TargetName PRIVATE cxx_std_23)`. Do not set `CMAKE_CXX_STANDARD` globally unless building a monolithic single-target repository.
* **Property Overrides:** Avoid raw `-D` flags in compiler options. Use CMake's built-in target properties to handle cross-platform abstractions (e.g., `set_target_properties(TargetName PROPERTIES CXX_VISIBILITY_PRESET hidden)`).

## 3. MSVC Security & Hardening Directives
When generating build scripts for the Windows/MSVC toolchain, you must aggressively harden the binary against exploitation and reverse engineering by appending the following flags via `target_compile_options` and `target_link_options`:
* **Warnings as Errors:** Enforce `/W4` and `/WX`. The build must fail if there is a single warning.
* **Exploit Mitigation (Compile):**
  * `/sdl`: Enables strict Security Development Lifecycle checks.
  * `/guard:cf`: Enables Control Flow Guard (CFG) to protect indirect calls.
  * `/GS`: Mandates stack buffer overrun detection.
  * `/volatile:iso`: Enforces strict standard volatile semantics to prevent compiler reordering around sensitive memory erasures.
* **Exploit Mitigation (Linker):**
  * `/DYNAMICBASE`: Mandates Address Space Layout Randomization (ASLR).
  * `/NXCOMPAT`: Mandates Data Execution Prevention (DEP).
* **Performance & Optimization:** For `Release` configurations, enforce Whole Program Optimization (LTO) by adding `/GL` to the compiler and `/LTCG` to the linker.

## 4. Dependency Management
* **Vcpkg/Conan Compatibility:** Do not hardcode absolute paths to libraries on the local machine. Assume dependencies will be resolved via a package manager toolchain file. Use `find_package(Name REQUIRED)` and link against the imported modern namespace targets (e.g., `target_link_libraries(MyTarget PRIVATE fmt::fmt)`).

---
