# Concurrency, Multithreading & Synchronization

This document dictates the architectural mandates for concurrent programming, thread lifecycle management, and data synchronization in modern C++ (C++20/C++23). The AI must prioritize dead-lock prevention, data-race immunity, and cooperative cancellation over raw manual thread management.

## 1. Thread Lifecycle & RAII
* **Mandate `std::jthread` (C++20):** The use of `std::thread` is strictly deprecated due to its unsafe lifecycle (forgetting to join/detach causes `std::terminate`). You must exclusively use `std::jthread`. It guarantees RAII behavior (auto-joining on destruction) and provides a cooperative cancellation mechanism via `std::stop_token`.
* **Detached Threads:** Calling `.detach()` is strictly forbidden. It creates zombie threads that leak memory and make graceful shutdowns impossible.
* **Cooperative Interruption:** Long-running loops inside threads must periodically check the `std::stop_token` (e.g., `while (!stop_token.stop_requested())`) to allow the main thread to cancel them gracefully.

## 2. Synchronization & Mutexes
* **No Manual Locking:** Explicit calls to `.lock()` and `.unlock()` on any mutex are strictly forbidden. You must use RAII wrappers to guarantee exception safety and release locks automatically.
* **`std::scoped_lock`:** Use `std::scoped_lock` (C++17) as the default choice for acquiring one or more mutexes. It completely prevents deadlocks when locking multiple mutexes simultaneously.
* **Reader-Writer Locks:** For data structures that are frequently read but rarely written to, use `std::shared_mutex` (C++17). Use `std::shared_lock` for concurrent readers and `std::unique_lock` for exclusive writers.
* **Condition Variables:** When using `std::condition_variable`, always pass a lambda predicate to the `.wait()` call to prevent the thread from proceeding due to spurious wakeups: `cv.wait(lock, [] { return data_ready; });`.

## 3. Race Conditions & Shared State
* **Immutable by Default:** The best way to prevent a data race is to not share mutable state. Favor message-passing (queues) and the Actor model over sharing complex mutable objects between threads.
* **Thread-Local Storage:** When a thread requires scratchpad memory or non-shared state, declare it as `thread_local` to avoid lock contention entirely.

## 4. Atomics & Lock-Free Paradigms
* **`std::atomic` over `volatile`:** The `volatile` keyword must never be used for thread synchronization. It does not prevent CPU reordering or guarantee atomic memory access. Always use `std::atomic<T>`.
* **Memory Ordering:** Default to `std::memory_order_seq_cst` (Sequential Consistency) for atomic operations. Only downgrade to `std::memory_order_acquire` and `std::memory_order_release` if profiled performance metrics strictly demand it, and the lock-free semantics are flawlessly architected.
* **Modern Primitives (C++20):** Utilize modern synchronization primitives instead of building custom spinlocks. Use `std::counting_semaphore`, `std::latch`, and `std::barrier` for coordinating complex multi-stage parallel pipelines.

## 5. Asynchronous Tasks
* **`std::async` & Futures:** For one-off background calculations returning a value, prefer `std::async` with the `std::launch::async` policy over spawning a manual thread. Handle the result using `std::future`.
* **Composed Pipelines (C++26):** For a *multi-stage* async pipeline (dependent steps, fan-out/fan-in across schedulers) rather than a single one-off calculation, prefer `std::execution` senders/receivers over chained `std::async`/`std::future` continuations — see [20_senders_receivers_cpp26.md](20_senders_receivers_cpp26.md) for structured-concurrency guarantees that eliminate a whole class of lifetime races.

---

## Code Generation Directives for AI Models

1. **No Implementation Comments:** `.cpp` files must contain zero comments. Synchronization boundaries and thread management must be self-evident through proper RAII scoping and descriptive lambda names.
2. **English Exclusivity:** All code constructs, variables, and literal strings must be natively written in English.
3. **Purity:** Thread-safe reader methods must be marked `const`. Return values from asynchronous tasks (`std::future`, `std::expected`) must be rigorously checked using `[[nodiscard]]`.