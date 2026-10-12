# ABI Interoperability, FFI & Boundary Security

This document dictates the architectural mandates for exposing C++ functionality to external languages (C#, Python, Lua, etc.) via Foreign Function Interfaces (FFI). The AI must prioritize strict ABI stability, opaque handles, and absolute exception containment.

## 1. The C-ABI Boundary Mandate
* **Strict `extern "C"`:** C++ name mangling varies between compilers and versions. Every function exposed to the outside world must be wrapped in `extern "C"` to guarantee a stable, predictable export name.
* **No C++ Types Across Boundaries:** You are strictly forbidden from passing or returning complex C++ STL types (`std::string`, `std::vector`, smart pointers) across the DLL boundary. Different memory allocators or standard library versions will cause heap corruption.
* **Primitive Types Only:** Only pass universally understood primitives (`std::uint32_t`, `double`, raw pointers) across the boundary. For strings, pass `const char*` and a `size_t` length.

## 2. Opaque Handles (The C-Pimpl)
* **Handle Inversion:** When an external language needs to manage a C++ object, do not expose the class. Instead, dynamically allocate the C++ object internally and return an opaque pointer (`void*` or a forward-declared struct pointer) to the caller.
* **Lifecycle Functions:** You must provide explicitly paired C-API functions for object creation and destruction (e.g., `MyEngine_Create(...)` returning `void*`, and `MyEngine_Destroy(void* handle)`). The C++ side must internally cast the `void*` back to the strong type and safely delete it.

## 3. Exception Sealing (The Barrier)
* **Zero Exceptions Leakage:** C++ exceptions (`throw`) traversing a C-ABI boundary result in undefined behavior and immediate process termination.
* **Catch-All Wrappers:** Every single exported `extern "C"` function must wrap its internal C++ logic in a strict `try/catch(...)` block. 
* **Error Codes:** If an internal exception is caught, the exported function must swallow it and return a standardized, universally understood error code (e.g., `HRESULT`, `NTSTATUS`, or a custom `enum` integer) to the caller.

## 4. Export Directives & Calling Conventions
* **`__declspec` Discipline:** In Windows environments, explicitly mark exported functions with `__declspec(dllexport)` during compilation and `__declspec(dllimport)` during consumption. Use a preprocessor macro tied to the CMake build target to toggle this automatically.
* **Calling Conventions:** Explicitly define the calling convention for exported functions (usually `__stdcall` or `__cdecl`) to prevent stack corruption when invoked via P/Invoke (C#) or `ctypes` (Python).

---
