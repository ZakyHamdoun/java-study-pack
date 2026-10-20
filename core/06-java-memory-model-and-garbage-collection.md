# Java Memory Model and Garbage Collection — New-Grad Interview Cheat Sheet

## 1. JVM memory areas: conceptual map

The Java language/JVM specifications define runtime areas and behavior, but not every physical layout detail of a particular HotSpot build.

| Area | Typical contents | Key point |
|---|---|---|
| Heap | Objects and arrays | Shared by threads; managed by GC in ordinary HotSpot configurations |
| Java thread stacks | Frames for method calls, local variables, operand stacks, return/linkage state | Each thread has its own stack; deep recursion can cause `StackOverflowError` |
| Metaspace / class metadata | Runtime class metadata in HotSpot's native memory | Not the same thing as the Java heap; class-loader leaks can retain metadata |
| Code cache | JIT-generated native machine code | Managed separately from the Java object heap |
| Native/direct memory | JNI/native allocations, direct buffers, runtime structures | Can exhaust process memory even when heap metrics look healthy |
| Per-thread structures | Thread-local state, native stacks, scheduling/runtime data | Creating huge numbers of platform threads has resource costs |

A **stack frame** is associated with a method invocation and stores method execution state. Local primitive values may be in registers/stack slots, and object references may be stored in locals; the object they reference is normally on the heap in the conceptual model. JIT escape analysis and scalar replacement mean physical allocation/placement can be optimized away, so do not treat the diagram as a literal layout guarantee.

### `OutOfMemoryError` vs `StackOverflowError`

- `OutOfMemoryError` can arise from Java heap exhaustion, metaspace/native memory exhaustion, direct-buffer exhaustion, inability to create native threads, and other resource limits.
- `StackOverflowError` usually means a thread's call stack cannot accommodate another frame, often due to runaway recursion.
- A heap dump helps inspect retained objects; native-memory tracking and process-level metrics matter when heap usage does not explain the failure.

## 2. Reachability and garbage-collection basics

GC normally reclaims objects that are no longer reachable from **GC roots** such as live thread stacks/registers, static references/class structures, and other VM-defined roots. The precise root set changes with runtime state and implementation.

- **Reachable objects remain live** from the collector's perspective, even if the application no longer “needs” them logically. A static cache or listener reference can therefore cause a memory leak.
- **Unreachable cycles are collectible.** Reference counting is not the sole basis of Java GC, so two objects only pointing at each other can still be collected if neither is reachable from a root.
- Garbage collection reclaims memory; it is not a deterministic resource-management mechanism. Use try-with-resources for files/sockets/database connections instead of waiting for GC.
- `System.gc()` is a request/hint to the runtime, not a guarantee of immediate/full collection; it may be ignored or configured to behave differently.

Reference types:

- **Strong** (ordinary references): keep objects reachable.
- **Soft**: may be cleared under memory pressure; not a reliable general-purpose cache policy because eviction timing is collector-dependent.
- **Weak**: cleared once keys/objects are no longer strongly reachable (subject to reachability processing); useful for some association/metadata patterns.
- **Phantom**: used with reference queues for post-mortem cleanup coordination; `get()` does not retrieve the referent. `Cleaner`/explicit resource management are often preferable for ordinary designs.

Cycles are not an issue by themselves; accidental roots, caches, static collections, unremoved listeners, class loaders, and `ThreadLocal` values are common retention issues.

## 3. How collectors reclaim memory

A simplified tracing GC may:

1. Find roots.
2. Trace reachable references.
3. Identify objects not reachable from roots.
4. Reclaim or compact/evacuate memory as appropriate.

Common algorithm terms:

- **Mark-sweep:** mark live objects, sweep dead objects. Can leave fragmentation.
- **Mark-compact:** move live objects together to reduce fragmentation, but moving/rewriting references has a cost.
- **Copying/evacuation:** copy live objects from a region/space to another; useful for high-allocation young generations.
- **Generational hypothesis:** most newly allocated objects die young. A generational collector treats young/old objects differently; remembered sets/card tables help track old-to-young references.
- **Stop-the-world (STW):** application threads pause for a GC phase. “Concurrent collector” does not mean no pauses at all.
- **Concurrent marking/relocation:** some GC work runs alongside application threads, consuming CPU and requiring barriers/coordination to preserve correctness.
- **Write/read barriers, card marking, remembered sets:** runtime mechanisms that help a collector discover/maintain reference relationships while the application runs.

The performance trade-off is generally among pause-time/latency, throughput, CPU overhead, memory footprint, and operational complexity. No collector is universally “best.”

## 4. HotSpot collectors: compare by workload

Java 27 release-specific facts matter: **G1 is now the default collector in all HotSpot environments** via JEP 523, and **compact object headers are enabled by default** via JEP 534. Explicit collector flags can still select another supported collector.

| Collector | General character | Useful when | Trade-offs / notes |
|---|---|---|---|
| Serial (`-XX:+UseSerialGC`) | Single-threaded GC work, generally STW | Small heaps, simple/single-core workloads, minimal GC worker overhead | Longer pauses as live data/work grows; doesn't exploit parallel GC workers |
| Parallel (`-XX:+UseParallelGC`) | Parallel STW collection; throughput-oriented | Batch/compute workloads that prioritize total work over short pauses | Longer pauses may be acceptable; GC uses multiple CPU cores |
| G1 (`-XX:+UseG1GC`) | Region-based, generational, parallel + mostly concurrent | General server/workstation default; seeks predictable pause goals with solid throughput | Has STW pauses and concurrent overhead; tune only after observing real behavior |
| Generational ZGC (`-XX:+UseZGC`) | Low-latency collector designed for very short pauses; modern ZGC is generational | Large heaps or services with strict latency/pause needs | Can trade some throughput/CPU for latency; measure on actual workload |
| Shenandoah | Concurrent low-pause collector in distributions that ship it | Latency-sensitive services where available/supported | Availability/defaults depend on JDK distribution/platform; not a universal Oracle HotSpot option |
| Epsilon (`-XX:+UseEpsilonGC`, where supported) | No-op collector; does not reclaim ordinary garbage | Testing, benchmarks, extremely constrained experiments with externally bounded allocation | Not for normal long-lived production workloads; heap exhaustion is expected |

Collector names and flags should be checked against the exact vendor/build. For interviews, explain the workload trade-off before quoting a flag.

### G1 basics

- Divides the heap into regions rather than one fixed contiguous young/old layout; regions are assigned young/old roles dynamically.
- Collects selected regions with the aim of meeting a pause-time target (a goal, not a hard real-time guarantee).
- Young collections evacuate live objects from selected regions; concurrent marking helps identify regions with reclaimable old-generation space.
- Large **humongous** objects can be costly and may contribute to fragmentation/collection pressure.
- Begin with defaults; measure before manually setting young-generation size parameters because they can interfere with G1's pause-time adaptation.

### ZGC basics

- Optimizes for low pause times and can support very large heaps.
- Modern ZGC is generational; do not repeat old guidance that ZGC is inherently non-generational or that it requires long STW full-heap compaction.
- “Sub-millisecond pause” is a design goal/typical claim, not a guarantee for every application/runtime/environment.

## 5. Garbage-collection troubleshooting

First ask:

1. Is the issue allocation rate, retained live set, fragmentation/large objects, native memory, or CPU contention?
2. Are pauses frequent, long, or both? What does GC logging show?
3. Does heap occupancy return to a stable level after collection, or does the live set keep growing?
4. Are pauses on the latency-critical path, and which percentiles (p95/p99/max) matter?

Useful tools/flags:

```bash
java -Xms2g -Xmx2g -Xlog:gc*:file=gc.log:time,uptime,level,tags -jar app.jar
jcmd <pid> GC.heap_info
jcmd <pid> GC.class_histogram
jcmd <pid> JFR.start name=profile settings=profile duration=60s filename=profile.jfr
jcmd <pid> GC.heap_dump /tmp/heap.hprof
```

Exact `jcmd` commands/options vary by JDK; use `jcmd <pid> help` on the target runtime. Heap dumps may contain sensitive data. `-Xmx` limits Java heap, not all memory used by the process. Diagnose with JFR, GC logs, heap dumps, class-loader analysis, native-memory tracking, and application metrics rather than reflexively increasing the heap.

Common causes:

- High allocation churn (short-lived temporary objects, boxing, excessive copying).
- Retained references (unbounded caches, static lists, listener registrations, `ThreadLocal` values, class-loader leaks).
- Large arrays/objects and large `byte[]` / buffers.
- Native/direct memory exhaustion independent of heap.
- Throughput/pause goal unsuitable for workload.

## 6. Java Memory Model (JMM): visibility and ordering

The JMM defines the semantics of interactions between threads, including which writes a read is allowed to observe. Wall-clock order alone is not a reliable substitute for a formal **happens-before** relationship.

Important happens-before edges include:

- Program order within one thread.
- Unlock of a monitor happens-before a subsequent lock of that same monitor.
- A volatile write happens-before a subsequent read of that same volatile field.
- `Thread.start()` happens-before actions in the started thread.
- All actions in a thread happen-before another thread successfully returns from `join()` on it.
- Correctly specified concurrency utilities define additional memory-consistency effects (for example, submission to an executor and task completion/future retrieval).

If A happens-before B, effects of A are visible/ordered before B under the JMM. If there is no suitable ordering/visibility guarantee, a data race can produce surprising values and reorderings; `sleep()` and “it works on my machine” do not fix it.

### `synchronized`

- Provides mutual exclusion for a monitor and creates memory visibility/order guarantees around unlock/lock of the same monitor.
- Instance synchronized method locks `this`; static synchronized method locks the `Class` object.
- The same monitor is reentrant: a thread may acquire it again while already owning it.
- A synchronized block should guard the complete invariant, not only an individual field access if several fields form one state change.

### `volatile`

- A volatile read sees a value ordered by the volatile synchronization rules; volatile accesses establish visibility and ordering.
- A single volatile read or write is atomic. `volatile long`/`double` accesses are also atomic under the JMM.
- `volatile` does **not** make compound read-modify-write operations atomic. `count++` is still read, add, write; use synchronization or an atomic class when increments can race.
- Volatile is useful for state flags and safe publication patterns, not a general replacement for locks around multi-step invariants.

```java
private volatile boolean running = true;
void stop() { running = false; }
void loop() {
    while (running) { doWork(); }
}
```

### Safe publication and `final`

- Publish shared objects through a happens-before mechanism: static initialization, a volatile reference, synchronized hand-off, thread-safe collection/queue, or other documented concurrency boundary.
- Correctly constructed objects with `final` fields get special initialization-safety guarantees, provided `this` did not escape during construction. These rules do not remove the need for safe publication or make mutable referenced objects thread-safe.
- Double-checked locking requires the reference to be `volatile` (or another correct publication design); otherwise it can expose a partially initialized object under the JMM.

## 7. Java 27 object headers and preview features

- **Compact Object Headers by Default (JEP 534):** Java 27 makes compact object headers the default HotSpot object-header layout. Oracle's release description says headers are reduced from 96 to 64 bits on 64-bit architectures, improving footprint/density/locality. Do not assume every JVM vendor/build uses identical object layouts.
- **G1 Default Everywhere (JEP 523):** Java 27 changes HotSpot's default collector selection across environments. Production should still be validated with real workload metrics.
- **Lazy Constants** are a Java 27 third-preview API; they aim to provide one-time-set immutable contents with JVM optimizations similar to `final` fields. Treat them as preview, not baseline language knowledge.

## 8. Common interview traps (True / False)

| # | Statement | Answer | Why |
|---:|---|:---:|---|
| 1 | Every Java object is guaranteed to occupy a literal heap allocation for its full lifetime. | **False** | JIT escape analysis/scalar replacement can optimize allocations away. |
| 2 | Java local variables always default to zero/null. | **False** | Definite-assignment rules apply to locals. |
| 3 | An unreachable object cycle can be collected. | **True** | Tracing GC uses reachability from roots, not simple reference counts. |
| 4 | GC guarantees prompt closing of files. | **False** | Use deterministic resource management. |
| 5 | `System.gc()` guarantees a full GC immediately. | **False** | It is a request/hint and may be ignored. |
| 6 | If heap use is low, a Java process cannot run out of memory. | **False** | Native/metaspace/direct-buffer/thread resources can be exhausted. |
| 7 | `StackOverflowError` often results from runaway recursion. | **True** | Each call consumes stack frame capacity. |
| 8 | `OutOfMemoryError` only means the Java heap is full. | **False** | Multiple memory/resource areas can fail. |
| 9 | Concurrent GC means the application never pauses. | **False** | Concurrent collectors still have pause phases. |
| 10 | A stop-the-world pause stops application threads participating in that VM. | **True** | That is the meaning of STW (details vary for some runtime threads). |
| 11 | More frequent minor/young collections always indicate a leak. | **False** | High allocation churn can produce frequent young collections with a stable live set. |
| 12 | G1 is the HotSpot default across all environments in Java 27. | **True** | JEP 523 changes this default. |
| 13 | Java 27 compact object headers are a language-level syntax feature. | **False** | They affect the HotSpot object layout. |
| 14 | `volatile int count; count++;` is atomic as a compound operation. | **False** | Increment is read-modify-write. |
| 15 | Volatile writes and later reads of the same field have JMM ordering/visibility guarantees. | **True** | That is a key volatile rule. |
| 16 | `Thread.sleep()` creates a happens-before edge with every thread. | **False** | It is not a synchronization mechanism. |
| 17 | Unlocking a monitor happens-before a later lock of the same monitor. | **True** | This is a core JMM synchronization edge. |
| 18 | Starting a thread safely publishes all earlier actions to that thread. | **True** | Actions before `start` happen-before actions in the started thread. |
| 19 | A thread's successful `join()` return is ordered after actions in the joined thread. | **True** | This is another specified happens-before edge. |
| 20 | `final` makes every object reachable from a field deeply immutable. | **False** | It restricts reassignment and provides initialization guarantees, not deep immutability. |
| 21 | An object can be safely published through a correct volatile-reference handoff. | **True** | Volatile ordering provides a publication edge for prior writes. |
| 22 | Soft references are a deterministic cache eviction strategy. | **False** | Clearing is memory-pressure/GC dependent. |
| 23 | Weak references are equivalent to manually expiring a cache entry after a timeout. | **False** | They follow reachability, not wall-clock policy. |
| 24 | Parallel GC is generally throughput-oriented. | **True** | It uses parallel workers but retains stop-the-world phases. |
| 25 | ZGC is designed for low pause times and is still a “no-pause” absolute guarantee. | **False** | The design targets very low pauses, not zero pauses under every condition. |
| 26 | Modern ZGC has generational operation. | **True** | Generational ZGC is the modern design; obsolete historical guidance can mislead. |
| 27 | Increasing `-Xmx` is always the right fix for memory growth. | **False** | A leak, native memory issue, or excessive churn needs diagnosis. |
| 28 | `-Xmx` caps all memory in the Java process. | **False** | It limits the Java heap only. |
| 29 | G1 tuning should usually begin with measuring logs/behavior and changing few parameters. | **True** | Many manual tuning flags can interfere with adaptive policy. |
| 30 | Correctly constructed `final` fields have special JMM initialization guarantees. | **True** | With conditions such as no `this` escape during construction. |

## Compact summary

- Know heap vs stack vs metaspace/code cache/native memory; heap is not the entire process footprint.
- GC reclaims unreachable objects from roots; cycles can be collected, and GC is not deterministic resource cleanup.
- Compare Serial (simple), Parallel (throughput), G1 (balanced/default in Java 27 HotSpot), ZGC (low latency), distribution-dependent Shenandoah, and Epsilon (no-op/testing).
- Understand reachability, generational GC, STW vs concurrent phases, allocation churn vs memory leaks, and diagnostic tools.
- JMM interview core: happens-before, `synchronized`, `volatile`, safe publication, atomic read-modify-write, and final-field guarantees.

## Official references

- [HotSpot GC Tuning Guide, Java 26](https://docs.oracle.com/en/java/javase/26/gctuning/index.html)
- [Available collectors, Java 26](https://docs.oracle.com/en/java/javase/26/gctuning/available-collectors.html)
- [JLS Chapter 17: Threads and Locks, Java SE 27](https://docs.oracle.com/javase/specs/jls/se27/html/jls-17.html)
- [Java 27 release notes](https://www.oracle.com/java/technologies/javase/27all-relnotes.html)
- [JEP 534: Compact Object Headers by Default](https://openjdk.org/jeps/534)
- [JEP 523: Make G1 the Default Garbage Collector in All Environments](https://openjdk.org/jeps/523)
