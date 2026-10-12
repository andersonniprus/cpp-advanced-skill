# C++26 Static Reflection (P2996)

This document dictates the architectural mandates for using C++26 static reflection to eliminate hand-written boilerplate (enum-to-string tables, per-field serialization, field-by-field comparison/hashing). Reflection is **experimental** at the time of writing: gate every use behind a feature check and never assume it compiles unconfirmed.

## 1. Mandatory Gating
* **Feature-Test Guard:** Wrap every reflection-dependent declaration behind `#if defined(__cpp_impl_reflection)` (or the project's `HAS_STATIC_REFLECTION` build macro), with a manual fallback path (hand-written or macro-generated) compiled otherwise. Never emit ungated reflection code — the target toolchain may not implement P2996 yet.
* **Confirm the Toolchain First:** Before generating reflection code, verify (or ask the user to confirm) that the active compiler ships an implementation of P2996-era static reflection. Do not silently assume a stable, universally-available feature.
* **Doxygen Disclosure Only:** If the feature is used, the `.hpp` `@brief` must note "C++26 static reflection, requires `__cpp_impl_reflection`" — this is documentation, not an implementation comment, and belongs in the header per [01_core_guidelines.md](01_core_guidelines.md).

## 2. Core Operators
* **`^^` (Reflection Operator):** Lifts a type, expression, namespace, or member into the reflection domain, producing a `std::meta::info` value. Example: `^^MyStruct`, `^^someVariable`.
* **`[: :]` (Splice Operator):** The inverse — lowers a `std::meta::info` back into the program domain wherever a type, expression, or identifier is grammatically expected. Splices compose with member access: `object.[: memberReflection :]` accesses the reflected member on `object`.

## 3. Key Metafunctions (`<meta>`)
* **`std::meta::nonstatic_data_members_of(^^T)`:** Returns the reflections of `T`'s non-static data members, in declaration order — the primary entry point for field-by-field code generation.
* **`std::meta::members_of` / `enumerators_of`:** Enumerate general members or, for an enum type, its enumerators — the basis for enum-to-string/string-to-enum generation.
* **`std::meta::identifier_of(info)` / `type_of(info)` / `template_of(info)`:** Extract the name, type, or originating template of a reflected entity for use in `constexpr` string building or further splicing.
* **`std::meta::access_context`:** Some queries require an explicit access context argument to honor member visibility (`private`/`protected`) correctly — do not assume public-only access when reflecting internal types.

## 4. Canonical Use Cases (Replacing Hand-Written Boilerplate)
* **Enum-to-String / String-to-Enum:** Replace X-macro tables with a `consteval` function iterating `enumerators_of(^^E)` to build a `std::array<std::string_view, N>` indexed by the enum's underlying value.
* **Automatic (De)Serialization:** Replace per-field JSON/binary marshalling with a templated function that iterates `nonstatic_data_members_of(^^T)`, splicing each member to read/write it generically. Preserve the project's `std::expected<T, ErrorEnum>` error-handling mandate ([01_core_guidelines.md](01_core_guidelines.md)) for malformed input, even in generated paths.
* **Structural Comparison / Hashing:** Generate `operator==`/`hash_value` by folding over reflected members instead of writing (and forgetting to update) a manual field list.
* **Struct-to-Tuple:** Convert a struct into a `std::tuple` of its member values for interop with tuple-based APIs (`std::apply`, structured bindings pipelines):

```cpp
template <typename T>
constexpr auto structToTuple(const T& value) {
    constexpr auto members = std::meta::nonstatic_data_members_of(^^T);
    return [&]<std::size_t... Index>(std::index_sequence<Index...>) {
        return std::make_tuple(value.[: members[Index] :]...);
    }(std::make_index_sequence<members.size()>{});
}
```

## 5. Expansion statements (C++26)

`template for` iterates a pack or a compile-time range at instantiation time, without an `index_sequence` lambda. Use it to walk `nonstatic_data_members_of` when the body is a statement, not an expression that must expand inside `make_tuple(...)`.

```cpp
template for (constexpr auto member : std::meta::nonstatic_data_members_of(^^T)) {
    hash = mix(hash, value.[: member :]);
}
```

Pack indexing and structured bindings that introduce a pack live in [02_modern_features.md](02_modern_features.md). Do not reintroduce recursive pack templates for a single index. Gate `template for` with the same reflection macro; fallback is the `index_sequence` pattern in §4.

## 6. Interop With Core Mandates
* Reflection-generated accessors are not exempt from the skill's baseline rules: mark generated read-only functions `const`, mark generating factory functions `[[nodiscard]]`, and route generated fallible paths through `std::expected` rather than exceptions.
* Prefer reflection over `consteval`-based string-hashing metaprogramming ([08_security_and_production.md](08_security_and_production.md)) only for structural introspection tasks (field iteration, enum mapping); keep the existing `consteval` FNV-1a/MurmurHash3 hashing approach for API-hashing and Anti-RE use cases, which are orthogonal to reflection.

## 7. Accuracy Discipline
* Never fabricate `std::meta` convenience functions beyond the documented P2996 core (`members_of`, `nonstatic_data_members_of`, `enumerators_of`, `identifier_of`, `type_of`, `template_of`, `access_context`). If the active compiler's `<meta>` surface diverges, prefer the documented core primitives and note the discrepancy in the Doxygen `@brief` rather than inventing an API. Do not invent the grammar of `template for` beyond the statement form above.