# Java Engineer Study Pack

## 1. Java Collections Framework

### 1.1. Core collection types

| Interface / type       | Main characteristics                               | When to use                                                        |
| ---------------------- | -------------------------------------------------- | ------------------------------------------------------------------ |
| `ArrayList`            | Resizable array; fast indexed access               | Default list when you need ordered elements and frequent reads     |
| `LinkedList`           | Doubly linked list; implements `List` and `Deque`  | Specialized deque operations; rarely the best general-purpose list |
| `ArrayDeque`           | Resizable circular-array deque                     | Best default for stacks and queues without concurrency             |
| `HashSet`              | Hash table; unique elements; no ordering guarantee | Fast membership checks and deduplication                           |
| `LinkedHashSet`        | Hash table plus insertion-order links              | Unique elements while preserving insertion order                   |
| `TreeSet`              | Balanced tree; sorted unique elements              | Sorted iteration, ranges, `floor`, `ceiling`                       |
| `HashMap`              | Hash table; key-value pairs                        | General-purpose key-value lookup                                   |
| `LinkedHashMap`        | Hash map plus linked ordering                      | Predictable iteration order or LRU-style caches                    |
| `TreeMap`              | Balanced tree; sorted keys                         | Ordered maps, range queries, nearest keys                          |
| `PriorityQueue`        | Heap; head is the smallest element by default      | Scheduling, top-k problems, repeatedly extracting min/max          |
| `ConcurrentHashMap`    | Concurrent hash-based map                          | Shared map with concurrent access                                  |
| `CopyOnWriteArrayList` | Reads use snapshots; writes copy the backing array | Read-heavy workloads with infrequent writes                        |

Interview rule of thumb: start with `ArrayList`, `HashMap`, `HashSet`, and `ArrayDeque`. Choose a more specialized structure when its behavior solves a specific requirement.

### 1.2. Time complexity of collections

Let \\(n\\) be the number of elements. These are typical complexities, not unconditional guarantees.

| Collection      | Access by index  | Search / contains         | Insert                              | Delete                              | Important caveat                                                       |
| --------------- | ---------------- | ------------------------- | ----------------------------------- | ----------------------------------- | ---------------------------------------------------------------------- |
| `ArrayList`     | \\(O(1)\\)       | \\(O(n)\\)                | Append: amortized \\(O(1)\\)        | End: \\(O(1)\\); middle: \\(O(n)\\) | Shifting elements makes middle operations expensive                    |
| `LinkedList`    | \\(O(n)\\)       | \\(O(n)\\)                | At a known node/end: \\(O(1)\\)     | At a known node/end: \\(O(1)\\)     | Finding the position still costs \\(O(n)\\)                            |
| `ArrayDeque`    | Not supported    | \\(O(n)\\)                | At either end: amortized \\(O(1)\\) | At either end: amortized \\(O(1)\\) | No null elements; no indexed access                                    |
| `HashSet`       | N/A              | Average \\(O(1)\\)        | Average \\(O(1)\\)                  | Average \\(O(1)\\)                  | Worst case can be \\(O(n)\\)                                           |
| `LinkedHashSet` | N/A              | Average \\(O(1)\\)        | Average \\(O(1)\\)                  | Average \\(O(1)\\)                  | Preserves insertion order                                              |
| `TreeSet`       | N/A              | \\(O(\log n)\\)           | \\(O(\log n)\\)                     | \\(O(\log n)\\)                     | Sorted order                                                           |
| `HashMap`       | N/A              | Average \\(O(1)\\) by key | Average \\(O(1)\\)                  | Average \\(O(1)\\)                  | Worst case can degrade; collision handling is implementation-dependent |
| `LinkedHashMap` | N/A              | Average \\(O(1)\\) by key | Average \\(O(1)\\)                  | Average \\(O(1)\\)                  | Maintains encounter order                                              |
| `TreeMap`       | N/A              | \\(O(\log n)\\) by key    | \\(O(\log n)\\)                     | \\(O(\log n)\\)                     | Keys remain sorted                                                     |
| `PriorityQueue` | Head: \\(O(1)\\) | \\(O(n)\\)                | \\(O(\log n)\\)                     | Head removal: \\(O(\log n)\\)       | Removing an arbitrary element is \\(O(n)\\)                            |

For `HashMap` and `HashSet`, average constant-time operations assume a reasonable hash distribution and well-behaved `hashCode()` and `equals()` implementations.

### 1.3. Choosing the right collection

| Requirement                       | Best starting point               | Why                                            |
| --------------------------------- | --------------------------------- | ---------------------------------------------- |
| Fast lookup by ID                 | `HashMap`                         | Average \\(O(1)\\) key lookup                  |
| Keep unique values                | `HashSet`                         | Efficient membership checks                    |
| Preserve insertion order          | `LinkedHashMap` / `LinkedHashSet` | Predictable iteration                          |
| Keep keys sorted                  | `TreeMap`                         | \\(O(\log n)\\) updates and sorted traversal   |
| Frequently access by index        | `ArrayList`                       | \\(O(1)\\) indexing                            |
| FIFO processing                   | `ArrayDeque`                      | Efficient queue operations                     |
| LIFO stack                        | `ArrayDeque`                      | Prefer over legacy `Stack`                     |
| Repeatedly extract minimum        | `PriorityQueue`                   | \\(O(\log n)\\) removal of the head            |
| Many readers, few writers         | `CopyOnWriteArrayList`            | Snapshot iteration without reader-side locking |
| Shared concurrent key-value store | `ConcurrentHashMap`               | Designed for concurrent operations             |
| Sorted range queries              | `TreeMap` / `TreeSet`             | `subMap`, `subSet`, `floor`, `ceiling`         |

### 1.4. `HashMap` details interviewers love

| Topic                           | Interview-ready answer                                                                                            |
| ------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| How does lookup work?           | Hash the key, locate a bucket, then compare candidate keys using equality                                         |
| `hashCode()` vs `equals()`      | Equal objects must have equal hash codes; equal hash codes do not imply equal objects                             |
| What happens during collisions? | Multiple keys share a bucket; Java can use tree bins for sufficiently populated buckets under suitable conditions |
| What is resizing?               | The table grows when its capacity/load threshold is exceeded, redistributing entries                              |
| Can it contain null?            | One null key and multiple null values are permitted                                                               |
| Is it thread-safe?              | No. Use a suitable concurrent collection or external synchronization                                              |
| Can a mutable object be a key?  | Yes, but changing fields involved in equality or hashing after insertion can break lookups                        |
| Is iteration order guaranteed?  | No                                                                                                                |

Common follow-up: Why must `equals()` and `hashCode()` be consistent? Because hash-based collections use the hash code to locate candidates before using equality to identify the key.

## 2. Garbage collection (GC)

Java manages memory automatically. The garbage collector identifies objects that are no longer reachable from GC roots and reclaims their memory. You generally do not free objects manually.

### 2.1. JVM memory basics

| Area                                | Stores                                                       | Interview point                               |
| ----------------------------------- | ------------------------------------------------------------ | --------------------------------------------- |
| Heap                                | Most objects and arrays                                      | Main area managed by GC                       |
| Young generation (where applicable) | Newly allocated objects                                      | Many short-lived objects die here             |
| Old generation (where applicable)   | Longer-lived objects                                         | Objects surviving collections may be promoted |
| Thread stacks                       | Method frames, local variables, references, primitive locals | Each thread has its own stack                 |
| Metaspace                           | Class metadata                                               | Uses native memory, not the Java heap         |
| Code cache                          | JIT-compiled machine code                                    | Managed separately from ordinary heap objects |

The generational layout is collector-dependent. For example, G1 uses young and old regions within the heap; not every modern collector uses the same organization.

### 2.2. Garbage collectors compared

| Collector   | Main characteristics                                              | When to consider it                                                                                   |
| ----------- | ----------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| Serial GC   | Single GC thread; stop-the-world collection                       | Small heaps, simple applications, limited CPU resources                                               |
| Parallel GC | Multiple GC threads; focuses on throughput                        | Batch jobs and applications where throughput matters more than pause consistency                      |
| G1 GC       | Region-based, generational, balances throughput and pause goals   | General-purpose server applications; often a good default starting point                              |
| ZGC         | Designed for very low pause times, including with large heaps     | Latency-sensitive services where long pauses are costly                                               |
| Shenandoah  | Concurrent reclamation and compaction designed to minimize pauses | Latency-sensitive workloads when supported by the chosen JDK distribution                             |
| Epsilon GC  | Does not reclaim memory                                           | Specialized testing, performance experiments, or short-lived processes with a known allocation budget |

Important: defaults and availability depend on JDK version, vendor, and runtime configuration. G1 is the usual default starting point on modern server-class HotSpot configurations, but verify the actual runtime.

### 2.3. GC vocabulary

| Term                  | Meaning                                                                                                         |
| --------------------- | --------------------------------------------------------------------------------------------------------------- |
| GC root               | A starting reference, such as a live thread stack reference or static reference, used to determine reachability |
| Reachability          | An object can be reached by following references from GC roots                                                  |
| Minor/young GC        | Collection focused on the young generation in generational collectors                                           |
| Major GC              | An ambiguous term; clarify which generation or operation the speaker means                                      |
| Full GC               | Broad heap collection, often more disruptive; exact behavior is collector-specific                              |
| Stop-the-world (STW)  | Application threads are paused while certain GC work happens                                                    |
| Concurrent collection | Some GC work happens while application threads continue running                                                 |
| Compaction            | Moves live objects to reduce fragmentation                                                                      |
| Memory leak           | Objects remain reachable even though the application no longer needs them                                       |
| Allocation rate       | Amount of memory allocated over time                                                                            |
| GC pause              | Time application threads are stopped for a GC phase                                                             |

### 2.4. How to choose a GC

| Scenario                                | Starting point                       | What to measure                                             |
| --------------------------------------- | ------------------------------------ | ----------------------------------------------------------- |
| Typical backend service                 | G1 GC                                | Throughput, allocation rate, pause percentiles              |
| Large batch processing                  | Parallel GC or G1                    | Jobs per second, total runtime, CPU usage                   |
| Strict latency requirements             | ZGC, Shenandoah, or G1 after tuning  | p99/p999 latency and pause distribution                     |
| Tiny application with limited resources | Serial GC may be suitable            | Memory overhead and total runtime                           |
| Frequent long pauses                    | Diagnose before switching collectors | Heap occupancy, allocation pressure, GC logs, live-set size |

Useful JVM flags for investigation:

```
-Xms512m
-Xmx2g
-Xlog:gc*
```

- `-Xms`: initial/minimum heap sizing.
- `-Xmx`: maximum Java heap size.
- `-Xlog:gc*`: enables detailed GC logging on modern HotSpot JDKs.

Interview trap: calling `System.gc()` does not guarantee an immediate garbage collection. And increasing `-Xmx` does not automatically fix a memory leak.

## 3. Java keywords and modifiers

### 3.1. Core language keywords

| Keyword                   | Purpose                                                                | Example / important detail                                                                           |
| ------------------------- | ---------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `class`                   | Declares a class                                                       | `class User {}`                                                                                      |
| `interface`               | Declares an interface                                                  | Defines a contract                                                                                   |
| `extends`                 | Inherits a class or extends an interface                               | A class can extend only one class                                                                    |
| `implements`              | Implements interfaces                                                  | A class can implement multiple interfaces                                                            |
| `new`                     | Creates an object or array                                             | `new ArrayList<>()`                                                                                  |
| `this`                    | Refers to the current instance                                         | `this.id = id;`                                                                                      |
| `super`                   | Accesses superclass constructors or members                            | `super()` must be the first statement in a constructor when used                                     |
| `static`                  | Declares class-level members                                           | Shared field or utility method                                                                       |
| `final`                   | Prevents reassignment, overriding, or inheritance depending on context | A final reference does not make its object immutable                                                 |
| `abstract`                | Declares an incomplete class or method                                 | Abstract methods have no implementation body                                                         |
| `void`                    | Method returns no value                                                | `void save()`                                                                                        |
| `return`                  | Exits a method and optionally returns a value                          | `return result;`                                                                                     |
| `if`, `else`, `switch`    | Conditional control flow                                               | Branching                                                                                            |
| `for`, `while`, `do`      | Loops                                                                  | Iteration                                                                                            |
| `break`, `continue`       | Alters loop or switch flow                                             | Exit or skip an iteration                                                                            |
| `try`, `catch`, `finally` | Exception handling                                                     | `finally` typically runs when leaving the try construct, with caveats such as abrupt JVM termination |
| `throw`                   | Throws an exception                                                    | `throw new IllegalArgumentException();`                                                              |
| `throws`                  | Declares possible checked exceptions                                   | Part of the method signature                                                                         |
| `instanceof`              | Tests runtime type compatibility                                       | Supports pattern matching in modern Java                                                             |
| `synchronized`            | Provides monitor-based mutual exclusion and memory visibility          | Protects a critical section                                                                          |
| `volatile`                | Provides visibility and ordering guarantees for a variable             | Does not make compound operations atomic                                                             |
| `transient`               | Excludes a field from default Java serialization                       | Applies to Java's serialization mechanism                                                            |
| `enum`                    | Declares an enum type                                                  | Fixed set of constants                                                                               |
| `record`                  | Declares a record class                                                | A restricted, concise carrier for data; record components generate accessors                         |
| `var`                     | Reserved type name for local-variable type inference                   | Not a general replacement for explicit types or valid for fields                                     |
| `yield`                   | Returns a value from a switch expression's block                       | Distinct from thread scheduling terminology                                                          |
| `assert`                  | Checks a development-time invariant                                    | Disabled by default unless assertions are enabled                                                    |

### 3.2. Access modifiers and `static` / `final`

| Modifier                      | Same class | Same package | Subclass in another package  | Unrelated class in another package |
| ----------------------------- | ---------- | ------------ | ---------------------------- | ---------------------------------- |
| `public`                      | Yes        | Yes          | Yes                          | Yes                                |
| `protected`                   | Yes        | Yes          | Yes, subject to access rules | No                                 |
| package-private (no modifier) | Yes        | Yes          | No                           | No                                 |
| `private`                     | Yes        | No           | No                           | No                                 |

| Modifier               | Effect                                                                | Common use                                      |
| ---------------------- | --------------------------------------------------------------------- | ----------------------------------------------- |
| `static`               | Member belongs to the class, not an individual instance               | Constants, factories, stateless utility methods |
| `final` on a variable  | Assignment allowed only once                                          | Stable references and constants                 |
| `final` on a method    | Prevents overriding                                                   | Preserve behavior in subclasses                 |
| `final` on a class     | Prevents subclassing                                                  | Classes such as `String`                        |
| `abstract` on a class  | Class cannot be directly instantiated                                 | Shared base behavior                            |
| `abstract` on a method | Subclasses must provide an implementation unless they remain abstract | API contracts                                   |
| `sealed`               | Restricts which classes may extend or implement a type                | Controlled inheritance hierarchies              |
| `non-sealed`           | Reopens inheritance under a sealed hierarchy                          | Permit further extension                        |

### 3.3. `final`, immutability, and `volatile`

| Concept                   | Guarantees                                                         | Does not guarantee                                       |
| ------------------------- | ------------------------------------------------------------------ | -------------------------------------------------------- |
| `final int x`             | `x` cannot be reassigned                                           | That every variable or field in the program is immutable |
| `final List<String> list` | The reference cannot be reassigned                                 | That elements cannot be added or removed                 |
| Immutable object          | Its observable state cannot change after construction              | Thread safety of unrelated mutable state                 |
| `volatile int x`          | Reads and writes have visibility and ordering guarantees           | Atomicity of `x++`                                       |
| `synchronized` block      | Mutual exclusion on the same monitor and happens-before visibility | Automatic deadlock prevention                            |
| `AtomicInteger`           | Atomic operations such as increment and compare-and-set            | Atomicity of arbitrary multi-object transactions         |

Example interview question: Is `volatile int counter; counter++;` thread-safe? No. Increment consists of reading, adding, and writing. Use `AtomicInteger.incrementAndGet()` or appropriate synchronization.

### 3.4. `==` versus `equals()`

| Comparison             | What it checks                                               | Example                                       |
| ---------------------- | ------------------------------------------------------------ | --------------------------------------------- |
| `==` on primitives     | Primitive values                                             | `3 == 3`                                      |
| `==` on references     | Object identity: whether references point to the same object | `a == b`                                      |
| `equals()`             | Logical equality as defined by the object's implementation   | `"java".equals(new String("java"))` is `true` |
| `Objects.equals(a, b)` | Null-safe equality comparison                                | Returns `true` when both are null             |

## 4. Sorting algorithms and Big-O complexity

Let \\(n\\) be the number of elements. For sorting, interviewers commonly ask about average/worst-case time, extra space, stability, and when the algorithm is useful.

### 4.1. Sorting algorithms comparison

| Algorithm      | Best time        | Average time     | Worst time       | Extra space                             | Stable?                           |
| -------------- | ---------------- | ---------------- | ---------------- | --------------------------------------- | --------------------------------- |
| Bubble sort    | \\(O(n)\\)\*     | \\(O(n^2)\\)     | \\(O(n^2)\\)     | \\(O(1)\\)                              | Yes, with standard implementation |
| Selection sort | \\(O(n^2)\\)     | \\(O(n^2)\\)     | \\(O(n^2)\\)     | \\(O(1)\\)                              | Usually no                        |
| Insertion sort | \\(O(n)\\)       | \\(O(n^2)\\)     | \\(O(n^2)\\)     | \\(O(1)\\)                              | Yes                               |
| Merge sort     | \\(O(n\log n)\\) | \\(O(n\log n)\\) | \\(O(n\log n)\\) | \\(O(n)\\)\*\*                          | Yes                               |
| Quicksort      | \\(O(n\log n)\\) | \\(O(n\log n)\\) | \\(O(n^2)\\)     | Typically \\(O(\log n)\\) average stack | Usually no                        |
| Heapsort       | \\(O(n\log n)\\) | \\(O(n\log n)\\) | \\(O(n\log n)\\) | \\(O(1)\\) auxiliary\*\*\*              | No                                |
| Counting sort  | \\(O(n+k)\\)     | \\(O(n+k)\\)     | \\(O(n+k)\\)     | \\(O(n+k)\\)                            | Can be stable                     |
| Radix sort     | \\(O(d(n+k))\\)  | \\(O(d(n+k))\\)  | \\(O(d(n+k))\\)  | Usually \\(O(n+k)\\)                    | Yes, with stable digit passes     |

\* Bubble sort achieves \\(O(n)\\) best-case time with an early-exit optimization. \*\*Standard array-based merge sort uses \\(O(n)\\) auxiliary storage. \*\*\*Heapsort's usual in-place implementation uses constant auxiliary storage, excluding input storage.

Here, \\(k\\) is the counting-sort key range or radix base, and \\(d\\) is the number of digits/passes in radix sort.

### 4.2. When to use each algorithm

| Algorithm      | Good fit                                               | Main drawback                                              |
| -------------- | ------------------------------------------------------ | ---------------------------------------------------------- |
| Bubble sort    | Learning and tiny educational examples                 | Poor performance on large inputs                           |
| Selection sort | Small inputs when minimizing swaps matters             | Always quadratic comparisons                               |
| Insertion sort | Small or nearly sorted arrays                          | Quadratic worst case                                       |
| Merge sort     | Stable sorting and predictable worst-case time         | Extra memory for arrays                                    |
| Quicksort      | Fast general in-memory partition sorting               | Bad pivot choices can produce quadratic time               |
| Heapsort       | Need worst-case \\(O(n\log n)\\) with in-place sorting | Not stable; locality can be worse than quicksort           |
| Counting sort  | Integers with a small, manageable key range            | Memory cost depends on the range                           |
| Radix sort     | Fixed-width integers or suitable strings               | Requires multiple passes and suitable digit representation |

### 4.3. Java sorting facts

| Java API                        | Interview detail                                                                     |
| ------------------------------- | ------------------------------------------------------------------------------------ |
| `Arrays.sort(int[])`            | Uses an implementation optimized for primitive arrays; commonly Dual-Pivot Quicksort |
| `Arrays.sort(Object[])`         | Stable sort with guaranteed \\(O(n\log n)\\) comparisons under the API contract      |
| `Collections.sort(list)`        | Delegates to list sorting; stable sorting is guaranteed                              |
| `list.sort(comparator)`         | Sorts using natural ordering or a provided comparator                                |
| `Comparator.comparing(...)`     | Builds a comparator from a key extractor                                             |
| `Comparator.thenComparing(...)` | Adds a secondary ordering criterion                                                  |

Example:

```
List<String> names = new ArrayList<>(
    List.of("Zoe", "Alex", "Mia")
);

names.sort(Comparator.naturalOrder());

List<String> users = List.of("Zoe", "Alex", "Mia");
```

`names.sort(...)` mutates the list being sorted. `List.of(...)` creates an unmodifiable list, so attempting to sort `users` in place throws `UnsupportedOperationException`.

### 4.4. Other complexity patterns worth memorizing

| Complexity       | Typical example                               |
| ---------------- | --------------------------------------------- |
| \\(O(1)\\)       | Array index access                            |
| \\(O(\log n)\\)  | Binary search in a sorted random-access array |
| \\(O(n)\\)       | Linear search                                 |
| \\(O(n\log n)\\) | Efficient comparison sorting                  |
| \\(O(n^2)\\)     | Two nested loops over the same input          |
| \\(O(2^n)\\)     | Enumerating all subsets                       |
| \\(O(n!)\\)      | Enumerating all permutations                  |

For binary search, the input must be sorted, and a random-access structure such as an array is preferable for the usual \\(O(\log n)\\) runtime.

## 5. Multithreading and concurrency

This is one of the highest-value areas to prepare for Java interviews. Know the difference between concurrency and parallelism, how synchronization works, and how to identify races and deadlocks.

### 5.1. Core concepts

| Concept          | Definition                                                | Example or key point                         |
| ---------------- | --------------------------------------------------------- | -------------------------------------------- |
| Process          | An executing program with its own process-level resources | A running JVM process                        |
| Thread           | A path of execution within a process                      | Multiple Java threads can share heap objects |
| Concurrency      | Multiple tasks make progress during overlapping periods   | Tasks take turns on a single CPU             |
| Parallelism      | Multiple tasks execute simultaneously                     | Tasks running on different CPU cores         |
| Race condition   | Result depends on timing/interleaving                     | Two threads incrementing a shared counter    |
| Critical section | Code that accesses shared state requiring coordination    | Updating an account balance                  |
| Mutual exclusion | Only one thread at a time enters a protected region       | `synchronized`                               |
| Atomicity        | An operation appears indivisible                          | `AtomicInteger.incrementAndGet()`            |
| Visibility       | One thread's writes become observable to another          | `volatile`, locks, happens-before            |
| Deadlock         | Threads wait indefinitely for one another's resources     | Opposite lock acquisition order              |
| Livelock         | Threads remain active but fail to make progress           | Repeatedly backing off and retrying          |
| Starvation       | A thread cannot obtain resources or CPU time sufficiently | Unfair lock contention                       |
| Context switch   | CPU changes the executing thread                          | Has scheduling overhead                      |

### 5.2. Ways to create and manage threads

| API                        | Purpose                                                                         | When to use                                                 |
| -------------------------- | ------------------------------------------------------------------------------- | ----------------------------------------------------------- |
| `Thread`                   | Represents a thread                                                             | Low-level thread management or simple demonstrations        |
| `Runnable`                 | Represents a task returning no result                                           | Tasks executed by a thread or executor                      |
| `Callable<V>`              | Represents a task returning a value and potentially throwing checked exceptions | Result-producing background tasks                           |
| `Future<V>`                | Represents a pending or completed result                                        | Retrieve a result, cancel a task, check completion          |
| `ExecutorService`          | Manages task execution and lifecycle                                            | General-purpose thread-pool management                      |
| `ScheduledExecutorService` | Runs tasks after delays or periodically                                         | Timers and scheduled background jobs                        |
| `CompletableFuture`        | Composes asynchronous stages                                                    | Pipelines, dependent computations, asynchronous composition |
| `ForkJoinPool`             | Uses work-stealing for suitable tasks                                           | Recursive divide-and-conquer work                           |
| Virtual threads            | Lightweight Java threads, available since Java 21                               | Large numbers of mostly blocking tasks                      |

Example:

```
ExecutorService executor =
    Executors.newFixedThreadPool(4);

try {
    Future<Integer> result =
        executor.submit(() -> 2 + 3);

    System.out.println(result.get()); // 5
} finally {
    executor.shutdown();
}
```

`Future.get()` blocks until the result is available. In production code, also consider timeouts, cancellation, exception handling, and appropriate executor sizing.

### 5.3. `synchronized` vs `volatile` vs atomic classes

| Mechanism       | Mutual exclusion                               | Visibility / ordering                                        | Compound atomic operations                                                                         |
| --------------- | ---------------------------------------------- | ------------------------------------------------------------ | -------------------------------------------------------------------------------------------------- |
| `synchronized`  | Yes, for the same monitor                      | Yes, through monitor unlock/lock                             | Yes, if the whole operation is inside the protected region                                         |
| `volatile`      | No                                             | Yes, for the volatile variable and associated ordering rules | No, not by itself                                                                                  |
| `AtomicInteger` | Uses atomic operations; may use CAS internally | Yes, according to its memory semantics                       | Yes, for supported operations such as increment                                                    |
| `ReentrantLock` | Yes                                            | Yes, through lock/unlock                                     | Yes, while holding the lock                                                                        |
| `LongAdder`     | Designed for scalable concurrent accumulation  | Yes, with its documented semantics                           | Supports concurrent addition, but not as one atomic snapshot increment operation for all use cases |

Example: a race condition.

```
class Counter {
    private int count = 0;

    void increment() {
        count++; // Not thread-safe
    }
}
```

A safe alternative for a simple counter:

```
AtomicInteger count = new AtomicInteger();

count.incrementAndGet();
```

Use `synchronized` or a lock when several operations must form one indivisible transaction.

### 5.4. Java Memory Model and `happens-before`

| Rule          | Meaning                                                                                           |
| ------------- | ------------------------------------------------------------------------------------------------- |
| Program order | Earlier actions in one thread happen-before later actions in that thread                          |
| Monitor rule  | Unlocking a monitor happens-before a subsequent lock of that same monitor                         |
| Volatile rule | A write to a volatile variable happens-before a subsequent read of that variable that observes it |
| Thread start  | Calling `Thread.start()` happens-before actions in the started thread                             |
| Thread join   | Actions in a thread happen-before another thread successfully returns from `join()` on it         |
| Transitivity  | If A happens-before B and B happens-before C, A happens-before C                                  |

The Java Memory Model explains when one thread is guaranteed to observe another thread's writes. Without an appropriate happens-before relationship, unsynchronized shared mutable state can produce unexpected results.

### 5.5. Thread lifecycle

| State           | Meaning                                                                     |
| --------------- | --------------------------------------------------------------------------- |
| `NEW`           | Created but not started                                                     |
| `RUNNABLE`      | Eligible to run or running; Java does not expose a separate `RUNNING` state |
| `BLOCKED`       | Waiting to acquire a monitor lock                                           |
| `WAITING`       | Waiting indefinitely for another action                                     |
| `TIMED_WAITING` | Waiting for a bounded time                                                  |
| `TERMINATED`    | Execution has finished                                                      |

Common transitions include:

- `start()` moves a new thread toward `RUNNABLE`.
- `Thread.sleep(...)` causes `TIMED_WAITING`; it does not release monitors already held.
- `Object.wait()` releases the corresponding monitor and waits to be notified, interrupted, or timed out.
- `join()` waits for another thread to finish.
- `interrupt()` requests interruption; code should cooperate by handling or propagating interruption appropriately.

### 5.6. Locks, coordination, and concurrent utilities

| Tool                | Purpose                                                | Important detail                                                                       |
| ------------------- | ------------------------------------------------------ | -------------------------------------------------------------------------------------- |
| `ReentrantLock`     | Explicit mutual exclusion                              | Supports `tryLock()`, interruptible acquisition, and optional fairness                 |
| `ReadWriteLock`     | Separate read and write locking                        | Useful when reads substantially outnumber writes                                       |
| `CountDownLatch`    | Wait until a fixed count reaches zero                  | Cannot be reset                                                                        |
| `CyclicBarrier`     | Let a group of threads meet at a synchronization point | Reusable after successful trips                                                        |
| `Semaphore`         | Limit concurrent access to a resource                  | Controls permits rather than ownership of a monitor                                    |
| `BlockingQueue`     | Thread-safe producer-consumer handoff                  | Can block on empty/full conditions                                                     |
| `ConcurrentHashMap` | Concurrent key-value access                            | Atomic methods such as `compute` help with per-key updates                             |
| `ThreadLocal`       | Separate values per thread                             | In thread pools, clear values when no longer needed to avoid stale state and retention |

### 5.7. Deadlock: how to prevent it

| Cause                                          | Prevention                                    |
| ---------------------------------------------- | --------------------------------------------- |
| Inconsistent lock ordering                     | Always acquire locks in a global order        |
| Holding locks during slow I/O                  | Avoid unnecessary lock duration               |
| Unbounded lock waiting                         | Consider timed `tryLock()` where appropriate  |
| Nested locking across multiple objects         | Reduce lock coupling or redesign shared state |
| Threads waiting while holding needed resources | Review wait conditions and release behavior   |

Interview distinction: `sleep()` does not release a monitor; `wait()` releases the monitor on which it is called.

### 5.8. Platform threads vs virtual threads

| Platform threads                                             | Virtual threads                                                              |
| ------------------------------------------------------------ | ---------------------------------------------------------------------------- |
| Typically mapped more directly to OS threads                 | Scheduled by the JVM over a smaller number of carrier threads                |
| More expensive to maintain in very large numbers             | Designed to make large numbers of blocking tasks practical                   |
| Useful for CPU-bound work and ordinary thread-pool workloads | Particularly useful for I/O-heavy, blocking workloads                        |
| Thread-pool sizing is often important                        | Do not use virtual threads as a way to make CPU-bound work inherently faster |

Virtual threads became a permanent feature in Java 21. They improve scalability for suitable blocking workloads; they do not remove the need for concurrency control, resource limits, or backpressure.

## 6. Java Generics

Generics let you write reusable, type-safe code by parameterizing classes, interfaces, and methods with types.

### 6.1. Basic syntax

| Concept                | Example                                 | Why it matters                                               |
| ---------------------- | --------------------------------------- | ------------------------------------------------------------ |
| Generic class          | `class Box<T> { T value; }`             | Reusable class with a type parameter                         |
| Generic interface      | `interface Repository<T> { T find(); }` | Type-safe abstraction                                        |
| Generic method         | `<T> T first(List<T> list)`             | Method declares its own type parameter                       |
| Bounded type           | `<T extends Number>`                    | Restricts the type argument                                  |
| Multiple bounds        | `<T extends Number & Comparable<T>>`    | Class bound first, followed by interface bounds              |
| Wildcard               | `List<?>`                               | List with an unknown element type                            |
| Upper-bounded wildcard | `List<? extends Number>`                | Accepts lists whose element type is `Number` or a subtype    |
| Lower-bounded wildcard | `List<? super Integer>`                 | Accepts lists whose element type is `Integer` or a supertype |
| Diamond operator       | `new ArrayList<>()`                     | Lets the compiler infer type arguments                       |
| Raw type               | `List list`                             | Avoid: bypasses much of generic type checking                |

Example:

```
class Box<T> {
    private final T value;

    Box(T value) {
        this.value = value;
    }

    T getValue() {
        return value;
    }
}

Box<String> box = new Box<>("hello");
String value = box.getValue();
```

The compiler ensures that a `Box<String>` exposes a `String`, without requiring a cast at the call site.

### 6.2. Upper and lower bounds: PECS

PECS stands for Producer Extends, Consumer Super.

| Type                     | Read values as | Add values of type                 | Common use                   |
| ------------------------ | -------------- | ---------------------------------- | ---------------------------- |
| `List<String>`           | `String`       | `String`                           | Known element type           |
| `List<?>`                | `Object`       | Only `null` can be added safely    | Read an unknown-typed list   |
| `List<? extends Number>` | `Number`       | Cannot safely add a non-null value | Read numbers from a producer |
| `List<? super Integer>`  | `Object`       | `Integer` and its subtypes         | Write integers to a consumer |

Consider this method:

```
static double sum(List<? extends Number> values) {
    double total = 0;

    for (Number value : values) {
        total += value.doubleValue();
    }

    return total;
}
```

You can pass a `List<Integer>`, `List<Double>`, or `List<Number>`. You cannot safely add an arbitrary `Number` to the input list because the actual list could be a `List<Integer>`.

For a consumer:

```
static void addDefaults(
        List<? super Integer> destination) {
    destination.add(10);
    destination.add(20);
}
```

This can accept `List<Integer>`, `List<Number>`, or `List<Object>`.

### 6.3. Type erasure

Java implements most generic type information through type erasure: type parameters are erased to their bounds or `Object`, and the compiler inserts casts where needed.

| Restriction or behavior                                                  | Explanation                                                                                    |
| ------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------- |
| `new T()` is illegal                                                     | The runtime does not have enough information to instantiate an arbitrary erased type parameter |
| `new T[10]` is illegal                                                   | Generic array creation is not type-safe under erasure                                          |
| `obj instanceof List<String>` is generally illegal                       | `List<String>` is not reifiable at runtime                                                     |
| `List<Integer>` and `List<String>` are not distinct runtime classes      | Both erase to `List`                                                                           |
| `List<int>` is illegal                                                   | Generics require reference types; use `Integer`                                                |
| Static fields cannot use a class's type parameter as a per-instance type | Static members belong to the class, not a particular parameterization                          |

### 6.4. Invariance, covariance, and contravariance

| Type relationship                                         | Is it valid? | Reason                               |
| --------------------------------------------------------- | ------------ | ------------------------------------ |
| `Integer` is a subtype of `Number`                        | Yes          | Ordinary class inheritance           |
| `List<Integer>` is a subtype of `List<Number>`            | No           | Generics are invariant               |
| `List<Integer>` can be passed to `List<? extends Number>` | Yes          | Upper-bounded wildcard               |
| `List<Number>` can be passed to `List<? super Integer>`   | Yes          | Lower-bounded wildcard               |
| `List<Object>` can be passed to `List<? super Integer>`   | Yes          | `Object` is a supertype of `Integer` |

Why is `List<Integer>` not a subtype of `List<Number>`? If it were, code could add a `Double` to a list that is supposed to contain only integers.

### 6.5. `Comparable` vs `Comparator`

| Feature            | `Comparable<T>`                   | `Comparator<T>`                              |
| ------------------ | --------------------------------- | -------------------------------------------- |
| Purpose            | Defines a type's natural ordering | Defines an external ordering                 |
| Main method        | `compareTo(T other)`              | `compare(T a, T b)`                          |
| Defined by         | The type itself                   | Separate object or lambda                    |
| Multiple orderings | Typically one natural ordering    | Many different orderings                     |
| Typical use        | Default sorting for a value type  | Sorting users by name, age, or another field |

Example:

```
record Employee(String name, int age) {}

Comparator<Employee> byAge =
    Comparator.comparingInt(Employee::age);

Comparator<Employee> byName =
    Comparator.comparing(Employee::name);

List<Employee> employees = new ArrayList<>();
employees.sort(byAge.thenComparing(byName));
```

### 6.6. Common generic interview questions

| Question                                                         | Short answer                                                                                                               |
| ---------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| Why use generics?                                                | Compile-time type safety, fewer casts, reusable APIs                                                                       |
| What is a bounded type parameter?                                | A type parameter constrained with `extends`                                                                                |
| Can a generic class have static methods?                         | Yes, but they must declare their own type parameters if needed                                                             |
| Can you overload methods using only different generic arguments? | Not if their signatures erase to the same signature                                                                        |
| What is a raw type?                                              | A generic type used without its type arguments                                                                             |
| What is heap pollution?                                          | A parameterized variable refers to an object that does not satisfy the variable's apparent type                            |
| What is `@SafeVarargs`?                                          | An annotation for certain varargs methods or constructors whose implementation is safe despite generic varargs limitations |
| What is the difference between `T extends X` and `? extends X`?  | `T` declares a named type parameter; `?` represents an unknown type                                                        |

## 7. Bonus topics that often distinguish strong candidates

These topics connect the tables above to practical Java engineering.

### 7.1. Exceptions

| Type                 | Checked at compile time? | Example                | Typical handling                                                                          |
| -------------------- | ------------------------ | ---------------------- | ----------------------------------------------------------------------------------------- |
| Checked exception    | Yes                      | `IOException`          | Catch or declare with `throws`                                                            |
| Unchecked exception  | No                       | `NullPointerException` | Fix the underlying programming error; handle where recovery makes sense                   |
| `Error`              | No                       | `OutOfMemoryError`     | Serious JVM or environment problem; generally not something to catch for routine recovery |
| `try-with-resources` | Compiler-managed cleanup | `try (var in = ...)`   | Automatically closes resources implementing `AutoCloseable`                               |

### 7.2. `String`, `StringBuilder`, and `StringBuffer`

| Type            | Mutable? | Thread-safe?                         | Use case                                         |
| --------------- | -------- | ------------------------------------ | ------------------------------------------------ |
| `String`        | No       | Safe to share as an immutable object | Text values and keys                             |
| `StringBuilder` | Yes      | No                                   | Efficient string construction within a thread    |
| `StringBuffer`  | Yes      | Synchronized methods                 | Legacy cases requiring synchronized mutable text |

Repeated concatenation inside a loop can create unnecessary intermediate objects. Prefer `StringBuilder` for repeated modifications in a single thread.

### 7.3. `equals()` and `hashCode()` contract

| Rule                                                                     | Why it matters                                                      |
| ------------------------------------------------------------------------ | ------------------------------------------------------------------- |
| If `a.equals(b)` is true, their hash codes must match                    | Hash-based collections must locate equal keys consistently          |
| Equal hash codes do not imply equal objects                              | Hash collisions are permitted                                       |
| `equals()` should be reflexive, symmetric, transitive, and consistent    | Collections rely on predictable equality                            |
| If a key's equality/hash fields change after insertion, lookups can fail | The object may be stored in a bucket inconsistent with its new hash |

### 7.4. Java Streams

| Operation    | Type         | Behavior                                               |
| ------------ | ------------ | ------------------------------------------------------ |
| `filter()`   | Intermediate | Keeps matching elements                                |
| `map()`      | Intermediate | Transforms elements                                    |
| `flatMap()`  | Intermediate | Maps each element to a stream and flattens the results |
| `sorted()`   | Intermediate | Sorts elements                                         |
| `distinct()` | Intermediate | Removes duplicates based on equality                   |
| `collect()`  | Terminal     | Accumulates a result                                   |
| `toList()`   | Terminal     | Produces an unmodifiable list in modern Java           |
| `reduce()`   | Terminal     | Combines elements into a result                        |
| `anyMatch()` | Terminal     | Tests whether any element matches; may short-circuit   |
| `count()`    | Terminal     | Counts elements                                        |

Streams are lazy until a terminal operation runs. Avoid relying on side effects inside stream operations, especially parallel streams.
