# C++26 Contracts (P2900)

This document dictates the architectural mandates for using C++26 language Contracts (`pre`, `post`, `contract_assert`) to formalize function boundaries. Contracts are **experimental**: gate every use and never let them replace mandatory runtime validation.

## 1. Mandatory Gating
* **Feature-Test Guard:** Wrap every contract-bearing declaration behind `#if defined(__cpp_contracts)` (language) — check `<version>` for the associated `__cpp_lib_contracts` library macro where relevant — with a fallback of a manual `contract_assert`-equivalent (`assert`/early-return) compiled otherwise.
* **Confirm the Toolchain and Build Level First:** Contract checking has configurable evaluation semantics (commonly exposed as build levels — e.g. *ignore*, *observe*, *enforce*). Before relying on a contract for anything user-visible, confirm with the user which semantic the target build uses; an *ignore*-level build compiles the check out entirely.

## 2. Syntax
* **`pre(condition)`:** Attached to a function declaration; checked (per the active evaluation semantic) before the function body executes.
* **`post(condition)` / `post(result: condition)`:** Attached to a function declaration; checked after the function body executes and before returning. The `result:` form binds the return value under a chosen name for use in the condition.
* **`contract_assert(condition);`:** A statement usable inside a function body for internal invariants that do not belong in the public `pre`/`post` contract of the signature.

```cpp
[[nodiscard]] std::size_t computeChecksum(std::span<const std::byte> data)
    pre(!data.empty())
    post(result: result != 0)
{
    contract_assert(data.size() <= kMaxChecksumInput);
    return foldBytes(data);
}
```

## 3. Contracts Complement, Never Replace, Guard Clauses
* **Public/Untrusted Boundaries:** Values crossing a security or public-API boundary must still be validated with the skill's mandatory early-return guard clauses ([01_core_guidelines.md](01_core_guidelines.md)) and `std::expected` error propagation. Never gate an externally-facing security check solely behind a `pre` contract — an *ignore*-level release build would silently drop the check.
* **Internal Invariants:** Reserve `pre`/`post`/`contract_assert` for invariants that exist to catch programmer error during development (documented preconditions between trusted internal callers), where being compiled out in release is an acceptable, intentional tradeoff.
* **Documentation Value:** Even when checking is disabled, a contract on a signature is executable documentation read by both humans and the compiler's diagnostics — prefer it over a Doxygen `@pre`/`@post` comment that can silently drift from the actual behavior.
* **`post` is a predicate:** A `post` condition must not mutate observable state. It reads the return value and `const` object state.

## 4. Violation Handling
* **Default Handler:** An unhandled contract violation invokes the implementation-defined default violation handler, which is not guaranteed to terminate on every evaluation semantic.
* **Security-Hardened Builds:** In builds following [08_security_and_production.md](08_security_and_production.md), pair `enforce` semantics with a custom violation handler that fails closed (logs and terminates) rather than relying on the default handler's unspecified behavior.
