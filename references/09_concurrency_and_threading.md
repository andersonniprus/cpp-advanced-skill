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

## 5. Lock-free reclamation (C++26)

A retired object must not be deleted while another thread still holds a pointer to it. Reference counting is the wrong tool when the read path must stay lock-free.

* **`std::hazard_pointer` (`<hazard_pointer>`):** A reader publishes the address it is about to use. A writer retires the object; reclamation waits until no hazard pointer protects it. Use for a singly linked structure with frequent readers and rare writers (lock-free stack, hash table bucket).
* **`std::rcu` (`<rcu>`):** Read-copy-update. Readers enter a read-side critical section; writers publish a new version and wait until pre-existing readers have finished before freeing the old one. Use when reads dominate and updates replace a whole snapshot.
* Both are C++26 and easy to misuse. Gate them. Fallback: `std::mutex` or `std::shared_mutex`. Do not hand-roll a hazard-pointer domain.
* A race audit that finds "delete while another thread may still load this pointer" should name hazard pointers or RCU only when the hot path is already lock-free and the toolchain has the header. Otherwise name a mutex.

## 6. Asynchronous Tasks
* **`std::async` & Futures:** For one-off background calculations returning a value, prefer `std::async` with the `std::launch::async` policy over spawning a manual thread. Handle the result using `std::future`.
* **Composed Pipelines (C++26):** For a *multi-stage* async pipeline (dependent steps, fan-out/fan-in across schedulers) rather than a single one-off calculation, prefer `std::execution` senders/receivers over chained `std::async`/`std::future` continuations — see [20_senders_receivers_cpp26.md](20_senders_receivers_cpp26.md) for structured-concurrency guarantees that eliminate a whole class of lifetime races.

---
