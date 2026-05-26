# Windows Security, Anti-RE & Production Readiness

This document dictates the architectural mandates for securing C++ binaries in the Windows Environment (Win32/NTAPI) against reverse engineering, memory exploitation, and dynamic analysis. The AI must prioritize hostile-environment resilience, stealth, and strict API discipline.

## 1. Windows Memory Safety & W^X Enforcement
* **Strict W^X (Write XOR Execute):** You are strictly forbidden from allocating or modifying memory with the `PAGE_EXECUTE_READWRITE` protection flag. Memory must be either writable OR executable, never both simultaneously. Use `VirtualProtect` or `NtProtectVirtualMemory` to transition memory from `PAGE_READWRITE` to `PAGE_EXECUTE_READ` only after all writes are finalized.
* **Sensitive Data Erasure:** Cryptographic keys, authentication tokens, and user credentials must not linger in RAM. You must explicitly zero out sensitive memory before the object's destruction using `RtlSecureZeroMemory` to prevent the MSVC compiler's dead-store elimination pass (`/O2`) from optimizing the erasure away.
* **Buffer Overflow Prevention:** Never use legacy C functions (`strcpy`, `sprintf`). Use safe Windows equivalents (`StringCchCopy`, `StringCchPrintf`) or, preferably, modern C++ `std::span` and `std::string_view` for bounded memory access.

## 2. Anti-Reverse Engineering (Anti-RE)
* **Dynamic API Resolution & Compile-Time Hashing:** Do not call sensitive WinAPI functions directly, as this leaves a blatant footprint in the Import Address Table (IAT). You must dynamically resolve sensitive APIs (like `VirtualAllocEx`, `WriteProcessMemory`) using custom implementations of `GetProcAddress` and `LoadLibrary`.
  * **Strict `consteval` Hashing:** Implement API hashing using strict `consteval` algorithms like FNV-1a or MurmurHash3.
  * **Zero Plaintext Footprint:** The hashed constants must be computed strictly at compile-time (`constexpr`). Under no circumstances should plaintext API names or DLL names appear in the compiled `.rdata` or `.data` sections.
* **Compile-Time Obfuscation:** Use `consteval` or `constexpr` template metaprogramming to XOR-encrypt strings at compile-time. Decrypt them at runtime strictly within the local stack scope where they are used, and zero them out immediately after.
* **Header Erasure:** For payloads or injected modules, erase the DOS and NT headers (`IMAGE_DOS_HEADER`, `IMAGE_NT_HEADERS`) from memory immediately after execution begins to thwart memory dumping tools like Scylla or Process Hacker.
* **TLS Callbacks:** Utilize Thread Local Storage (TLS) callbacks to execute security checks, anti-debugging routines (e.g., checking PEB `BeingDebugged` flag), and environment validations *before* the main entry point (`main` or `DllMain`) is reached.


## 3. MSVC Compilation & Linker Directives
* **Symbol Stripping & RTTI:** Ensure the architecture does not rely on Run-Time Type Information (RTTI). `dynamic_cast` and `typeid` are strictly forbidden, as they leak class names into the `.rdata` section. Code must assume compilation with `/GR-`.
* **Exceptions:** Minimize the use of C++ exceptions (`throw`/`catch`) as they generate `.pdata` and `.xdata` unwinding tables that aid reverse engineers. Use `std::expected` and `NTSTATUS` codes for control flow.
* **Direct Syscalls (When Applicable):** To bypass user-mode hooks placed by EDRs in `ntdll.dll`, prefer generating or utilizing direct syscall stubs for critical NTAPI functions instead of calling the exported functions.

---

## Code Generation Directives for AI Models

1. **No Implementation Comments:** `.cpp` files must contain zero comments. Obfuscation techniques or WinAPI security boundaries must be self-evident through their function names and structural isolation.
2. **English Exclusivity:** All code constructs, variables, and literal strings must be natively written in English.
3. **Purity:** Methods that validate security states but do not mutate them must be explicitly marked `const`. Security check return values (`NTSTATUS`, `HRESULT`, `std::expected`) must be rigorously enforced using `[[nodiscard]]`.