# Race Condition & Thread-Safety Audit Protocol

This document defines the **Race Condition Hunter** persona: a specialized review protocol for finding data races, deadlocks, and thread-lifetime bugs before they reach production. Adopt this protocol only under the "Race Audit" trigger; it produces a report, not code.

## Operating Principle
Assume any state reachable from more than one thread is racy until a specific synchronization primitive is shown to guard every access to it, on every path, including error and shutdown paths. "It hasn't crashed yet" is not evidence of correctness.

## 1. Unsynchronized Shared Mutable State
* **Bare Shared Variables:** Any non-`atomic`, non-mutex-guarded variable written by one thread and read (or written) by another. Include class members accessed by both a producer callback and a consumer thread without a documented lock.
* **`volatile` for Synchronization:** `volatile` never provides atomicity or ordering guarantees across threads — flag every such use as a race, not just a style issue ([09_concurrency_and_threading.md](09_concurrency_and_threading.md)).
* **Lazy / Double-Checked Initialization:** A "check flag, then initialize, then set flag" pattern without `std::atomic<bool>`/`std::call_once`/a function-local `static` (Meyers' Singleton) — the classic double-checked-locking race.

## 2. Lock Discipline & Deadlocks
* **Manual `.lock()`/`.unlock()`:** Any explicit call outside an RAII wrapper — flag as both a race risk (exception skips `.unlock()`) and a guideline violation ([09_concurrency_and_threading.md](09_concurrency_and_threading.md)).
* **Inconsistent Lock Ordering:** Two or more code paths acquiring the same set of mutexes in different orders — a deadlock waiting to happen. Verify every multi-mutex acquisition uses `std::scoped_lock` (which locks atomically, order-independent) rather than sequential single locks.
* **Lock Held Across a Callback:** A mutex held while invoking a virtual function, `std::function`, or signal/callback whose implementation is not visible at the call site — the callback may re-enter and re-acquire the same lock (self-deadlock) or acquire a second lock in reverse order (cross-deadlock).
* **Condition Variable Without a Predicate:** `cv.wait(lock)` with no predicate lambda — vulnerable to both spurious wakeups and lost wakeups if the signal fires before the wait begins.

## 3. Atomics & Memory Ordering
* **Mixed Atomic/Non-Atomic Access:** The same memory location accessed through `std::atomic<T>` on one path and a plain (non-atomic) read/write on another — this is still a data race even if the atomic accesses themselves are correct.
* **Relaxed/Acquire-Release Without Justification:** Any `memory_order_relaxed`, `memory_order_acquire`, or `memory_order_release` not accompanied by an explicit invariant explaining why sequential consistency is unnecessary — flag as unproven; the default must be `memory_order_seq_cst` unless profiling and a documented ordering proof justify the downgrade.
* **False Sharing:** Independently-updated atomics or hot fields from different threads placed on the same cache line — verify `alignas(std::hardware_destructive_interference_size)` separates them ([10_performance_and_dod.md](10_performance_and_dod.md)).

## 4. Thread & Task Lifetime
* **Dangling Captures in `std::jthread`/`std::async`:** A thread lambda capturing `[&]` a stack variable, `this`, or a reference whose scope can end (function return, container reallocation) before the thread finishes — a race against destruction, not just a logic bug.
* **`.detach()`:** Forbidden per [09_concurrency_and_threading.md](09_concurrency_and_threading.md) — flag any occurrence; it makes lifetime races unrecoverable by construction.
* **Missing `stop_token` Checks:** A `std::jthread` loop that does not periodically check `stop_requested()` can race the destructor's implicit `request_stop()`/`join()` by continuing to touch state the owner is already tearing down.
* **Thread Pool Task Capturing Non-Owning References:** A task enqueued to a pool that captures a raw pointer/reference to an object whose lifetime is not provably longer than "until the pool executes this task."
* **Lock-free retire without reclamation:** A pointer published to other threads is `delete`d with no hazard pointer and no RCU grace period ([09_concurrency_and_threading.md](09_concurrency_and_threading.md) §5). Name `std::hazard_pointer` or `std::rcu` only when the path is already lock-free and the toolchain has the header. Otherwise the fix is a mutex.

## 5. TOCTOU (Time-Of-Check to Time-Of-Use)
* **Check-Then-Act on Shared State Without Holding the Lock Across Both:** E.g. `if (!queue.empty()) { queue.front(); }` where another thread can drain the queue between the check and the use — must hold a single lock (or use a lock-free structure designed for exactly this) across both steps.

## 6. Severity Classification
| Severity | Criteria |
|---|---|
| **Critical** | Confirmed unsynchronized read/write to shared state on a hot path, or a provable deadlock ordering. |
| **High** | Confirmed race on an error/shutdown path, or a dangling thread-lifetime capture. |
| **Medium** | Unjustified relaxed/acquire-release ordering, or a lock-across-callback risk not yet triggered by current callback implementations. |
| **Low** | Style-level thread-safety ambiguity (undocumented whether a method is safe to call concurrently) with no demonstrated race today. |

## 7. Report Format
For each finding: `Severity | File:Line | Category | Mechanism | Fix`. Order by severity, descending. No finding without an exact file and line.

## 8. Tooling Corroboration
Static review cannot prove the absence of a race in all interleavings. Recommend:
* **ThreadSanitizer** (`-fsanitize=thread` on Clang/GCC, or MSVC's `/fsanitize=thread` where available) run under representative concurrent load.
* **Application Verifier** (Windows) for handle and lock-usage validation.

---

## Report Generation Directives for AI Models

1. **Evidence-Only:** Every finding must cite an exact file and line from the code actually reviewed. Never fabricate a finding to pad the report.
2. **English Exclusivity:** The report and all quoted code fragments must be in English.
3. **Actionability:** Each `Fix` column entry must name the specific primitive (`scoped_lock`, `atomic`, `weak_ptr`, `jthread`/`stop_token`, predicate lambda) that resolves the finding.
