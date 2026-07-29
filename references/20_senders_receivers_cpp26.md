# C++26 Senders/Receivers & `std::execution` (P2300)

This document dictates the architectural mandates for using the C++26 `std::execution` senders/receivers framework to build composed, structured-concurrency asynchronous pipelines. It supersedes hand-rolled callback graphs and manual `std::jthread`/`std::future` orchestration for exactly that use case — composed, multi-stage async work — not for a single simple background loop.

## 1. Mandatory Gating
* **Feature-Test Guard:** Confirm `std::execution` availability via `<version>` before emitting it. Be aware `__cpp_lib_execution` historically named the C++17 parallel-algorithm execution-policy feature (`<execution>` header, `std::execution::par`/`seq`/`unseq`) — verify against the specific compiler's documented macro for the senders/receivers library rather than assuming the name, and fall back to `std::jthread`/`std::async` composition ([09_concurrency_and_threading.md](09_concurrency_and_threading.md)) otherwise.

## 2. Core Vocabulary
* **Sender:** A lazy description of asynchronous work — constructing a sender does not start it. Composable via pipe-style algorithm adaptors.
* **Receiver:** The completion callback for a sender, with three channels: `set_value`, `set_error`, `set_stopped` — corresponding to success, failure, and cancellation.
* **Operation State:** Produced by `execution::connect(sender, receiver)`; represents the concrete, non-movable resource for one execution. `execution::start` begins it.
* **Scheduler:** A lightweight handle to an execution context (thread pool, GPU queue, single-threaded event loop) obtained via `execution::schedule(scheduler)`, used to place work on a specific context.

## 3. Composable Algorithms
* **`execution::then(sender, f)`:** Runs `f` on the value(s) produced by `sender`'s success channel — the async equivalent of `std::optional::transform` ([14_optional_and_monadic_ops.md](14_optional_and_monadic_ops.md)).
* **`execution::let_value(sender, f)`:** Runs `f` on the value(s) produced by `sender`, where `f` returns a new sender to chain into — the async equivalent of `and_then`.
* **`execution::when_all(senders...)`:** Runs multiple senders concurrently and completes when all complete, propagating the first error if any fails.
* **`execution::on(scheduler, sender)` / `execution::schedule(scheduler)`:** Places work on a specific execution context.
* **`execution::sync_wait(sender)`:** Blocks the calling thread until the sender completes — the sanctioned bridge back into synchronous code (e.g. a `main()` or a legacy call site); never poll or spin-wait on a sender's completion manually.
* **`execution::start_detached(sender)`:** Starts a sender without a caller waiting on its result. Use sparingly and only when the sender's own completion path owns any resources it touches — this is the one place a lifetime race can still be reintroduced if used carelessly.

```cpp
auto pipeline = execution::schedule(pool.get_scheduler())
    | execution::then(loadPayload)
    | execution::let_value(validateAndParse)
    | execution::then(persist);

auto result = execution::sync_wait(std::move(pipeline));
```

## 4. Structured Concurrency and Race Elimination
* **Bounded Lifetime by Construction:** A composed sender's nested work is bound to the parent sender's completion — the parent does not complete until every child operation state has finished. This eliminates the "dangling capture in a detached async callback" class of finding from [17_race_condition_audit.md](17_race_condition_audit.md) §4, because there is no way for the pipeline's completion to be observed before its nested work has finished.
* **Preferred Fix for Lifetime Races:** When a Race Audit finds a `std::jthread`/`std::async` lambda capturing a stack reference/`this` that can outlive its target, and the target codebase confirms C++26 `std::execution` support, propose restructuring the async graph as a sender pipeline (`when_all`/`let_value`) instead of adding ad hoc synchronization — the structural guarantee is stronger than a manually-audited lock.

## 5. When to Stay on `std::jthread`
* A single, simple background loop bound to RAII lifetime with cooperative cancellation via `stop_token` ([09_concurrency_and_threading.md](09_concurrency_and_threading.md) §1) remains the correct default. Reach for `std::execution` specifically when the work is a *pipeline* of dependent async steps, needs to run across heterogeneous schedulers (CPU pool, GPU queue), or requires `when_all`-style fan-out/fan-in — not as a blanket replacement for every thread in the codebase.

---

## Code Generation Directives for AI Models

1. **No Implementation Comments:** `.cpp` files must contain zero comments. Each pipeline stage must be a descriptively named function or short lambda passed to `then`/`let_value`.
2. **English Exclusivity:** All code constructs, variables, and literal strings must be natively written in English.
3. **Purity:** Prefer stages that are free functions or `const` member functions with no hidden shared-state mutation; any stage that must mutate shared state still requires the synchronization discipline of [09_concurrency_and_threading.md](09_concurrency_and_threading.md).
