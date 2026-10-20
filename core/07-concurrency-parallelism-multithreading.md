# Concurrency, Parallelism, and Multithreading — New-Grad Interview Cheat Sheet

## 1. Concurrency vs parallelism

- **Concurrency**: multiple tasks make progress during overlapping periods; work may take turns on one core.
- **Parallelism**: multiple tasks execute at the same instant on different execution resources/cores.
- **Multithreading**: a programming mechanism for overlapping execution flows. Threads can be concurrent without being parallel.
- **CPU-bound work** needs computation; parallelism can help if there are enough cores and the algorithm splits well. **I/O-bound work** spends time waiting for network/files/database; scalable concurrency can improve throughput without reducing the time of each individual operation.
- Speedup is bounded by serial work, contention, scheduling/combining overhead, memory bandwidth, and hardware. More threads do not necessarily mean more throughput.

## 2. Thread creation and lifecycle

- `Thread.start()` schedules a new thread of execution. Calling `run()` directly is just a normal method call on the current thread.
- A thread cannot be started twice.
- State enum: `NEW`, `RUNNABLE`, `BLOCKED`, `WAITING`, `TIMED_WAITING`, `TERMINATED`. There is no separate public `RUNNING` state; `RUNNABLE` can include executing or ready-to-run threads.
- `join()` waits for a thread to finish. Interruption is cooperative; `interrupt()` sets/delivers interruption status and causes certain blocking methods to throw `InterruptedException`.
- If a method catches `InterruptedException` but cannot propagate it, commonly restore the status with `Thread.currentThread().interrupt()` after performing any required cleanup. Do not swallow cancellation silently.
- Daemon platform threads do not keep the JVM alive by themselves. Virtual threads are daemon threads and also do not prevent shutdown.

## 3. Platform threads and virtual threads

### Platform threads

A platform thread is closely backed by an OS thread. It suits CPU-intensive work and general execution but has more resource overhead, so large thread-per-request designs can hit native thread/stack limits.

### Virtual threads (final since Java 21)

Virtual threads are lightweight `java.lang.Thread` instances scheduled by the JVM on a smaller set of carrier/platform threads. When a virtual thread blocks in supported operations, it can often unmount and free the carrier for other virtual threads.

- Best fit: many concurrent tasks that mostly wait on I/O (HTTP, JDBC, files, service calls), with a straightforward thread-per-task programming style.
- Not intended to make CPU-bound code execute faster. CPU-bound throughput is still limited by cores.
- Do not normally pool virtual threads to limit their count; create one per task. Limit scarce downstream resources (e.g. DB connections, remote API quota) with semaphores/bulkheads/connection pools.
- Virtual threads do not remove the need for backpressure, timeouts, rate limits, cancellation, and error handling.
- Thread-local state can be costly when there are very many virtual threads. Be deliberate with `ThreadLocal` and inherited thread-local variables.
- As of Java 24, synchronized code no longer pins virtual threads merely because they block while holding a monitor (JEP 491); Java 26 also improves the case of virtual threads waiting for another thread to run a class initializer. Native/foreign calls and certain implementation-specific situations still deserve measurement.

```java
// Thread-per-task with virtual threads (Java 21+)
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    Future<String> a = executor.submit(() -> fetch("https://service-a"));
    Future<String> b = executor.submit(() -> fetch("https://service-b"));
    return combine(a.get(), b.get());
}
```

This example assumes `fetch` is defined and that checked exceptions are handled by the surrounding method. `ExecutorService` is `AutoCloseable` in modern Java, so try-with-resources coordinates shutdown. Consider timeouts and cancellation in real systems.

## 4. Race conditions and synchronization

A **race condition** occurs when correctness depends on uncontrolled timing/interleaving. A **data race** is a JMM-specific situation involving conflicting accesses to the same variable by multiple threads, at least one write, without a sufficient happens-before ordering.

```java
class Counter {
    private int count;
    void increment() { count++; } // not thread-safe
    int get() { return count; }
}
```

`count++` has read-modify-write steps. Two threads can read the same old value and overwrite each other's increments (lost update). It also has visibility/order issues if unsafely shared.

Ways to protect state:

- `synchronized` / monitor lock around the whole invariant.
- `ReentrantLock` or other `Lock` implementations when explicit lock operations/features are required; always release in `finally`.
- `volatile` for visibility/order on a single variable, not compound atomic updates.
- `AtomicInteger`, `AtomicReference`, `LongAdder` for suitable atomic/contended counters/state; understand their semantics.
- Immutable objects and message passing (blocking queues, actor-like ownership) to reduce shared mutable state.

```java
class Counter {
    private final AtomicInteger count = new AtomicInteger();
    void increment() { count.incrementAndGet(); }
    int get() { return count.get(); }
}
```

Use `LongAdder` for high-contention statistical counters when you mostly need aggregate reads; it is not a drop-in replacement where every intermediate `sum()` must be an atomic snapshot or a unique increment return value is required.

## 5. `synchronized`, monitors, `wait`, and `notify`

- Every Java object can act as a monitor lock. `synchronized` acquisition is reentrant.
- Unlocking a monitor happens-before a later lock of that same monitor.
- `wait()` must be called while owning the monitor. It releases that monitor while waiting and reacquires it before returning/throwing.
- `sleep()` pauses the current thread for a time but **does not release any monitor it holds**.
- `notify()` wakes one waiting thread (choice is not a fairness guarantee); `notifyAll()` wakes all waiters to contend to reacquire the monitor.
- Always test a wait condition in a loop, not an `if`, because of spurious wakeups and because another thread may change the condition before the awakened thread reacquires the lock.

```java
synchronized (lock) {
    while (!ready) {
        lock.wait();
    }
    consumeSharedState();
}
```

Prefer higher-level concurrency utilities over hand-written `wait`/`notify` unless you understand the monitor protocol fully.

## 6. `volatile` and the Java Memory Model

The core question is not just “is this assignment atomic?” but “what synchronizes this thread with that one, and what happens-before edge makes the write visible?”

- `volatile` read/write access has a defined visibility/order relationship for that field.
- A volatile flag can tell another thread to stop, but a multi-field invariant may require a lock.
- `final` does not make an arbitrary mutable object thread-safe.
- `Thread.start`, successful `join`, locks, volatile fields, and concurrency utilities establish important happens-before edges.
- `Thread.sleep`, `yield`, and thread priority do not establish general visibility guarantees.

## 7. Executors and task coordination

### Executors

- `Executor`: separates task submission from execution.
- `ExecutorService`: adds lifecycle management and `submit`/`Future`-style task handling.
- `ScheduledExecutorService`: delayed/repeated task scheduling.
- `Future`: represents a result/completion; `get()` blocks and wraps task failure in `ExecutionException`; cancellation is cooperative/best-effort.
- `CompletableFuture`: composes asynchronous stages (`thenApply`, `thenCompose`, `thenCombine`, `handle`, `exceptionally`). Non-async continuations may run on the thread completing the prior stage; `*Async` overloads use an executor (often the common ForkJoinPool if one is not supplied). Be explicit about execution context for blocking/latency-sensitive work.
- Always shut down owned executors or use an appropriate lifecycle pattern. Know rejection, queueing, pool sizing, and cancellation behavior.

### Pool sizing heuristics

- CPU-bound fixed pool often starts near available processor count; benchmark and account for other work.
- I/O-bound pool size depends on waiting ratio and resource limits, but unbounded pools/queues can cause overload. Virtual threads simplify the thread-per-task side but do not remove limits of databases, sockets, memory, or downstream services.
- For thread pools, understand core/max size, keep-alive, queue type/capacity, rejection policy, and whether work is allowed to pile up.

### Coordination utilities

| Utility | What it solves |
|---|---|
| `CountDownLatch` | One-shot gate: wait until N events complete |
| `CyclicBarrier` | Fixed group of parties meet at a reusable barrier |
| `Phaser` | Flexible/reusable phased coordination with changing parties |
| `Semaphore` | Limit concurrent access to N permits/resources |
| `BlockingQueue` | Producer-consumer handoff with blocking/bounded capacity |
| `CompletableFuture` | Dependency graph of asynchronous results |
| `ForkJoinPool` | Work-stealing pool for recursively split computation; common pool used by some parallel APIs |

## 8. Locks, atomics, lock-free, and ABA

- `ReentrantLock` supports explicit lock/unlock, optional fairness, interruptible/timed acquisition, and `Condition`s. Fairness has overhead and is not a universal throughput win.
- `ReadWriteLock` may help read-heavy workloads, but reader/writer coordination overhead can make it slower than a simple mutex for small critical sections.
- `StampedLock` includes optimistic reads but has more complex semantics and is not reentrant; use only with a clear need.
- Atomics commonly rely on compare-and-set (CAS). CAS checks the current value and updates only if it still equals the expected value.
- The **ABA problem**: a value can change A → B → A, so a CAS looking only at equality sees A and may miss the intermediate change. Version stamps or other designs may be needed in algorithms affected by ABA.
- **Lock-free** means system-wide progress is guaranteed despite some threads stalling; **wait-free** gives a per-operation progress bound. Neither simply means “uses no `synchronized` keyword.”

## 9. Concurrent collections

- `ConcurrentHashMap`: thread-safe map with highly concurrent retrieval/update design; does not allow null keys/values; iterators are weakly consistent, not fail-fast snapshots. Compound invariants involving multiple keys may require additional coordination.
- `ConcurrentLinkedQueue`: non-blocking queue; does not block when empty/full (unbounded), so use explicit backpressure elsewhere.
- `BlockingQueue` implementations (e.g. `ArrayBlockingQueue`, `LinkedBlockingQueue`) are useful for producer-consumer pipelines and bounded buffering.
- `CopyOnWriteArrayList`: excellent for read/traversal-heavy, rare-update workloads; iterators see an array snapshot.
- Concurrent collection safety does not automatically make objects stored inside them thread-safe. A `ConcurrentHashMap<String, List<X>>` does not make each inner `List` safe for simultaneous mutation.

## 10. Deadlock, starvation, and livelock

- **Deadlock**: a set of threads wait forever for resources held by each other.
- Four classic necessary conditions (Coffman): mutual exclusion, hold-and-wait, no preemption, circular wait.
- Prevent common lock deadlocks by imposing a global lock acquisition order, minimizing nested locks, using timeouts where appropriate, or replacing shared state with message passing.
- **Starvation**: one task/ thread is continually denied needed resources/progress.
- **Livelock**: threads remain active and respond to each other but make no useful progress.
- A timeout or interrupt can improve recovery, but does not prove an algorithm is deadlock-free.
- `jstack`/thread dumps and JFR are useful when investigating blocked threads, lock contention, and pool starvation.

## 11. Structured concurrency in Java 27

Java 27 includes **Structured Concurrency as a seventh preview** (JEP 533). The idea is to treat a group of related concurrent subtasks as one lexical unit of work so their lifetime, cancellation, failure, and observability are managed together. It complements—not replaces—virtual threads and executor APIs.

Because the API remains preview, signatures/behavior can still evolve. Use the Java 27 JEP/API documentation and the appropriate preview flags before using it; do not memorize preview signatures as permanent stable APIs. For an interview, explain the design goal: task structure mirrors program structure, with clear parent/child lifetimes and coordinated cancellation/error handling.

## 12. Common interview traps (True / False)

| # | Statement | Answer | Why |
|---:|---|:---:|---|
| 1 | Concurrency requires more than one CPU core. | **False** | Tasks can make overlapping progress on one core by interleaving. |
| 2 | Parallelism means executing multiple operations at the same instant. | **True** | That is the usual distinction. |
| 3 | Calling `Thread.run()` starts a new thread. | **False** | `start()` schedules another execution flow; `run()` is an ordinary method call. |
| 4 | A `Thread` instance can be started repeatedly after termination. | **False** | A thread can be started at most once. |
| 5 | `RUNNABLE` means the thread is definitely executing on a core right now. | **False** | It may be ready or running. |
| 6 | `sleep()` releases all monitors held by the sleeping thread. | **False** | It does not release monitors. |
| 7 | `Object.wait()` releases the monitor whose `wait` method was called. | **True** | It releases that monitor while waiting and reacquires it before return. |
| 8 | `wait()` must be invoked while holding the monitor. | **True** | Otherwise `IllegalMonitorStateException` is thrown. |
| 9 | A wait condition should generally be checked in a `while` loop. | **True** | Spurious wakeups and condition changes require rechecking. |
| 10 | `notify()` guarantees the highest-priority waiter runs next. | **False** | It does not provide that scheduling guarantee. |
| 11 | `volatile int x; x++` is atomic. | **False** | The compound increment is not atomic. |
| 12 | A volatile write happens-before a subsequent read of the same volatile field. | **True** | This is a core JMM guarantee. |
| 13 | `Thread.sleep()` is a memory barrier that publishes all writes. | **False** | It does not establish general happens-before. |
| 14 | Interruption is commonly a cooperative cancellation signal. | **True** | Code needs to respond appropriately. |
| 15 | If `InterruptedException` is caught and not propagated, restoring interrupt status is often appropriate. | **True** | It preserves the cancellation signal for callers/frameworks. |
| 16 | Virtual threads always make CPU-heavy calculations faster. | **False** | They improve scalability of blocking tasks, not raw CPU speed. |
| 17 | Virtual threads are suitable for high-concurrency, mostly blocking I/O tasks. | **True** | This is their primary fit. |
| 18 | Virtual threads remove database connection limits. | **False** | Downstream resource limits remain. |
| 19 | It is generally recommended to pool virtual threads just to limit the number of virtual threads. | **False** | Use thread-per-task; limit scarce resources separately. |
| 20 | Java 21 finalized virtual threads. | **True** | They are no longer a preview feature. |
| 21 | Java 27 structured concurrency is a finalized stable API. | **False** | It is a seventh preview in Java 27. |
| 22 | `ConcurrentHashMap` accepts null keys and values. | **False** | It rejects both. |
| 23 | `ConcurrentHashMap` makes a multi-step business invariant across several keys automatically atomic. | **False** | A compound invariant may need additional coordination. |
| 24 | A `ConcurrentHashMap` makes mutable values stored inside it thread-safe. | **False** | Value objects need their own safety/ownership strategy. |
| 25 | A bounded blocking queue can provide backpressure. | **True** | Producers can block/fail when capacity is reached. |
| 26 | `Future.get()` can block while waiting for a result. | **True** | It is a blocking retrieval operation. |
| 27 | `CompletableFuture` async callbacks always run on a dedicated thread pool. | **False** | Without an explicit executor, async methods commonly use the common pool. |
| 28 | A CAS loop can be affected by the ABA problem. | **True** | The observed value may return to A after intermediate changes. |
| 29 | Any algorithm without explicit locks is lock-free. | **False** | Progress guarantees are formal properties, not syntax. |
| 30 | Deadlock prevention can use a global lock acquisition order. | **True** | It prevents circular-wait patterns when followed consistently. |
| 31 | Starvation and deadlock mean exactly the same thing. | **False** | Starvation denies a task progress; deadlock is a circular/waiting state with no progress. |
| 32 | `CopyOnWriteArrayList` is usually optimal for frequent writes. | **False** | Writes copy the backing array. |
| 33 | Thread-safe collections automatically make compound iteration/update workflows atomic. | **False** | Their contract usually covers individual operations, not arbitrary workflows. |
| 34 | A synchronized method on an instance locks the `Class` object. | **False** | Instance synchronized methods lock `this`; static ones lock the Class object. |
| 35 | Virtual threads can be created with `Thread.startVirtualThread(...)` or a virtual-thread executor. | **True** | Both are supported modern APIs. |

## Compact summary

- Concurrency is overlapping progress; parallelism is simultaneous execution. Match concurrency strategy to CPU-bound vs I/O-bound work.
- Know `start` vs `run`, interruption, thread states, `wait` vs `sleep`, and the monitor/happens-before rules.
- `synchronized` protects invariants; `volatile` provides visibility/order for a field but does not make compound updates atomic. Know atomics, CAS, and ABA.
- Use executors, futures, latches, semaphores, and concurrent collections instead of reinventing coordination primitives.
- Virtual threads are scalable, lightweight threads for many blocking tasks; they do not speed up CPU work or expand downstream resource quotas. Structured concurrency is preview in Java 27.
- Know deadlock vs starvation vs livelock, lock ordering, backpressure, pool starvation, and the practical value of thread dumps/JFR.

## Official references

- [Thread API, Java SE 26](https://docs.oracle.com/en/java/javase/26/docs/api/java.base/java/lang/Thread.html)
- [Virtual Threads — dev.java / Oracle](https://docs.oracle.com/en/java/javase/26/core/virtual-threads.html)
- [Java concurrency utilities package summary](https://docs.oracle.com/en/java/javase/26/docs/api/java.base/java/util/concurrent/package-summary.html)
- [ConcurrentHashMap API, Java SE 26](https://docs.oracle.com/en/java/javase/26/docs/api/java.base/java/util/concurrent/ConcurrentHashMap.html)
- [JLS Chapter 17: Threads and Locks, Java SE 27](https://docs.oracle.com/javase/specs/jls/se27/html/jls-17.html)
- [JEP 444: Virtual Threads](https://openjdk.org/jeps/444)
- [Java 27 release notes](https://www.oracle.com/java/technologies/javase/27all-relnotes.html)
