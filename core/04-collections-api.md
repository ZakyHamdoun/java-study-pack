# Java Collections Framework — New-Grad Interview Cheat Sheet

## 1. Collection types: choose by contract, not habit

The main abstraction tree:

- `Iterable<E>`: can provide an iterator.
- `Collection<E>`: root abstraction for element groups; extended by `List`, `Set`, and `Queue` (and `Deque`).
- `Map<K,V>` is separate: maps keys to values and is not a subtype of `Collection`.
- `SequencedCollection`, `SequencedSet`, and `SequencedMap` (Java 21+) express a defined encounter order, with first/last/reversed operations where appropriate.

| Need | Common choice | Why / caveat |
|---|---|---|
| Indexed access, append-heavy list | `ArrayList` | Contiguous reference array; fast indexing/cache locality; middle insert/remove shifts elements |
| Queue or stack at both ends | `ArrayDeque` | Resizable circular array; generally preferred over legacy `Stack` or `LinkedList` for ordinary deque use; rejects null |
| Known node/iterator-heavy sequential list | `LinkedList` | Doubly-linked nodes; indexed operations still O(n), with pointer/allocation overhead |
| Unique items, no iteration-order promise | `HashSet` | Hash-table backed; expected O(1) basic operations if hashes disperse well |
| Unique items in insertion order | `LinkedHashSet` | Hash set plus linked encounter order |
| Unique sorted items | `TreeSet` | Tree-based ordering; typically O(log n); ordering determines set equivalence |
| Key lookup by hash | `HashMap` | Expected O(1); order unspecified; allows one null key and null values |
| Key lookup plus stable iteration order | `LinkedHashMap` | Insertion order by default; supports access-order mode useful for LRU-style designs |
| Sorted keys / range queries | `TreeMap` | O(log n), navigation/range views; comparator/natural ordering matters |
| Keys are enum constants | `EnumMap` / `EnumSet` | Compact, array/bit-oriented implementation specialized for one enum type |
| Thread-safe high-concurrency map | `ConcurrentHashMap` | Concurrent retrieval/update design; rejects null key and values |
| Read-mostly list shared among threads | `CopyOnWriteArrayList` | Iterators use snapshots; every write copies the backing array, so bad for write-heavy use |
| Priority scheduling by comparator | `PriorityQueue` | Heap; `peek` O(1), insert/remove head O(log n); iteration is not sorted |

Complexities are typical implementation behavior, not a guarantee for every possible user-defined comparator/hash behavior.

## 2. `ArrayList` vs `LinkedList` — a classic trap

| Operation | `ArrayList` | `LinkedList` |
|---|---:|---:|
| `get(i)` | O(1) | O(n) |
| append / addLast | amortized O(1) | O(1) |
| insert/remove at index i | O(n) because shifting | O(n) to find node; O(1) once at node |
| memory/locality | compact array of references | node objects plus links, more pointer chasing |
| queue/deque ends | can be inefficient at front | O(1) at both ends, but with greater overhead |

Do not assert “LinkedList insertion is always O(1).” Finding an indexed insertion location is usually O(n). In real systems, `ArrayList` often outperforms `LinkedList` for traversal due to locality and lower allocation overhead.

## 3. `HashMap` internals: buckets, collisions, resize

### Mental model

A `HashMap` maintains a table (array) whose elements are buckets. A key's hash helps choose a bucket; keys whose hashes map to the same bucket are collisions. Each mapping stores the key, value, hash, and link/tree information.

Typical OpenJDK implementation details (useful for interviews, **not API guarantees**):

1. Capacity is a power of two. For a non-null key, implementation spreads high hash bits into lower bits (often `h ^ (h >>> 16)`) because the bucket index uses a mask similar to `(capacity - 1) & hash`.
2. `hashCode()` narrows the bucket candidates; `equals()` identifies whether a key already exists among candidates. Equal keys must have equal hash codes.
3. Collisions in a bucket are stored as a linked list; sufficiently large bins can be converted to balanced tree bins under implementation-specific thresholds (commonly treeify threshold 8, but treeification is deferred while the table is small—commonly below capacity 64). Removal can convert bins back.
4. With default load factor 0.75, threshold is approximately `capacity × loadFactor`. Exceeding it triggers resize, allocating a larger table and redistributing entries. Resize is relatively expensive but amortized over many inserts.
5. Typical `get`/`put` expected complexity is O(1) with good hash distribution. Collision-heavy behavior can be much worse; tree bins are intended to reduce severe collision costs.

**Do not promise exact internal thresholds as a Java specification guarantee.** Source changes between JDK implementations/releases are possible. The public contract guarantees mappings/semantics, not bucket layout.

```java
Map<String, Integer> counts = new HashMap<>();
counts.merge("apple", 1, Integer::sum);
counts.computeIfAbsent("banana", ignored -> 0);

// Capacity planning when expected mapping count is known:
Map<String, User> users = HashMap.newHashMap(10_000); // since Java 19
```

### `equals` / `hashCode` contract for keys

- If `a.equals(b)` is true, `a.hashCode() == b.hashCode()` must be true.
- The reverse is not required: same hash does not imply equal objects.
- A hash key's equality/hash-relevant state should not be mutated while stored. If the hash changes, the entry can remain in the old bucket and become hard to retrieve/remove.
- `null` key is supported by `HashMap` and `HashSet` (which is backed by a map), but not by `ConcurrentHashMap`.

```java
record UserId(long value) {} // good immutable key shape
Map<UserId, String> byId = new HashMap<>();
byId.put(new UserId(7), "Ada");
```

Records can make good value keys if their components themselves obey stable equality/hash semantics.

## 4. Set and Map nuances

- `HashSet<E>` is backed by a `HashMap<E, Object>`-style structure; set elements are represented as keys.
- `LinkedHashMap` maintains a linked ordering structure as well as hash lookup; default is insertion order, while access-order can move recently accessed entries and support cache policies.
- `TreeMap`/`TreeSet` use natural ordering or a `Comparator`, usually a red-black tree with O(log n) search/insert/remove.
- For a sorted map/set, two keys that compare as zero are treated as the same key/element for sorted-collection operations even if `equals` is false. A comparator consistent with `equals` avoids confusing behavior.
- `IdentityHashMap` deliberately uses reference identity (`==`) rather than normal equality; use only when identity semantics are intended.
- `WeakHashMap` uses weak references for keys; a key can disappear when it is no longer strongly reachable elsewhere. It is not a deterministic eviction cache.
- `EnumMap` keys must all come from a single enum type and generally perform very well.

## 5. Ordering, iteration, and Java 21 sequenced collections

- `HashMap` and `HashSet` make no iteration-order guarantee. Never write tests or business logic that relies on their incidental iteration order.
- `LinkedHashMap`/`LinkedHashSet` preserve a defined linked order; `TreeMap`/`TreeSet` expose sorted order.
- Java 21 added sequenced collection interfaces and methods such as `getFirst`, `getLast`, `addFirst`/`addLast` where supported, `reversed()`, and sequenced-map first/last entry operations. This models a defined encounter order without assuming all ordered collections are sorted.
- A `PriorityQueue` exposes the least element (under its comparator) at the head; iterating it does **not** return fully sorted order. Repeated `poll()` does.

## 6. Unmodifiable vs immutable vs fixed-size

| Construction | What it means | Common trap |
|---|---|---|
| `List.of(a, b)` | Unmodifiable list; rejects null | Elements themselves may still be mutable |
| `List.copyOf(source)` | Unmodifiable snapshot/copy contract; rejects null | Shallow copy, not deep immutability |
| `Collections.unmodifiableList(list)` | Read-only wrapper view over the supplied list | Changes through another reference to the backing list remain visible |
| `Arrays.asList(array)` | Fixed-size list backed by the array; `set` works | `add`/`remove` unsupported; array/list element changes reflect both ways |
| `Stream.toList()` | Unmodifiable result list | Do not assume it returns a particular concrete type |
| `new ArrayList<>(source)` | New mutable shallow copy | Nested mutable elements are shared |

“Unmodifiable” restricts collection operations through that reference; “immutable” is stronger and often includes stable element state. A shallow copy does not recursively copy contained objects.

## 7. Iterators, fail-fast behavior, and concurrency

- Fail-fast iterators may throw `ConcurrentModificationException` after a structural modification not performed through that iterator.
- This is best-effort bug detection, **not** synchronization and not a guarantee in a data race. Never make correctness depend on receiving `ConcurrentModificationException`.
- The normal `ArrayList`, `HashMap`, etc. are not thread-safe for unsynchronized concurrent mutation.
- `Collections.synchronizedList` and similar wrappers synchronize individual operations, but compound actions and iteration still need a correct external synchronization protocol (usually synchronize on the wrapper during iteration).
- Concurrent collection iterators have different contracts: `ConcurrentHashMap` iterators are weakly consistent; `CopyOnWriteArrayList` iterators read a snapshot.
- Use `ConcurrentHashMap` instead of synchronizing an entire `HashMap` when its concurrency semantics fit, but understand that multi-step invariants may still require coordination.

## 8. Useful `Map` methods and their semantics

- `get(key) == null` does not distinguish “no mapping” from “mapped to null” in a map that permits null values. Use `containsKey` if that distinction matters.
- `putIfAbsent`, `computeIfAbsent`, `compute`, `merge`, `replace`, and `remove(key, value)` simplify atomic-ish map operations. For `ConcurrentMap` implementations, their concurrency contracts are stronger than doing a separate `get` then `put`.
- Do not mutate the same map from a remapping function in ways the method prohibits; keep mapping callbacks simple and side-effect-light.
- `Map.of(...)` / `Map.ofEntries(...)` create unmodifiable maps and reject null keys/values and duplicate keys.

## 9. Performance interview prompts

- Need fast random access? Prefer `ArrayList` over `LinkedList`.
- Need frequent end insert/remove? `ArrayDeque` is often the best queue/stack default.
- Need order by key/range queries? `TreeMap`/`TreeSet`.
- Need unique elements but don't care order? `HashSet`.
- Need predictable insertion order and good lookup? `LinkedHashMap`.
- Need concurrency? Pick concurrent structures based on access pattern; do not assume a synchronized wrapper makes a multi-operation algorithm atomic.
- Need memory/performance claims? State expected vs worst case and benchmark the real workload. Big-O omits allocation, constant factors, cache locality, contention, and hash quality.

## 10. Common interview traps (True / False)

| # | Statement | Answer | Why |
|---:|---|:---:|---|
| 1 | `Map` extends `Collection`. | **False** | It is a separate abstraction. |
| 2 | `HashMap` guarantees insertion order. | **False** | Order is unspecified. |
| 3 | `LinkedHashMap` can preserve insertion order. | **True** | That is its default ordering mode. |
| 4 | `TreeMap` is typically O(log n) for lookup. | **True** | It uses a balanced search tree. |
| 5 | `ArrayList.get(i)` is typically O(1). | **True** | It indexes a backing array. |
| 6 | `LinkedList.get(i)` is O(1). | **False** | It traverses from an end to the index. |
| 7 | Appending to `ArrayList` is always worst-case O(1). | **False** | A resize copies elements; append is amortized O(1). |
| 8 | `HashSet` uses a `HashMap`-style backing structure in OpenJDK. | **True** | Set elements are typically stored as map keys. |
| 9 | Equal keys may legally have different hash codes. | **False** | Equal keys must have equal hashes. |
| 10 | Different keys must have different hash codes. | **False** | Collisions are permitted. |
| 11 | Mutating a key's hash-relevant fields can break lookups. | **True** | Its current hash may no longer select its storage bucket. |
| 12 | HashMap uses `equals` but does not use `hashCode`. | **False** | Both are central to normal lookup. |
| 13 | A bucket is always just one entry. | **False** | Multiple colliding keys may share a bucket. |
| 14 | HashMap's exact treeification threshold is guaranteed by the public API. | **False** | Bucket layout and thresholds are implementation details. |
| 15 | `HashMap` supports a null key and null values. | **True** | It allows both, with one null key mapping. |
| 16 | `ConcurrentHashMap` supports null keys and null values. | **False** | It rejects both. |
| 17 | A `PriorityQueue` iterator returns elements in sorted order. | **False** | Only head ordering is guaranteed; repeated `poll` yields priority order. |
| 18 | `Arrays.asList` has normal resizable-list add/remove behavior. | **False** | It is fixed-size and array-backed. |
| 19 | `Collections.unmodifiableList` makes the backing list immutable. | **False** | Other references can mutate the backing list. |
| 20 | `List.copyOf` is a deep copy of every element. | **False** | It is shallow. |
| 21 | `List.of` accepts null elements. | **False** | It rejects null. |
| 22 | Fail-fast iteration is a thread-safety guarantee. | **False** | It is best-effort bug detection. |
| 23 | Synchronized wrappers automatically make every compound operation atomic. | **False** | Multi-step sequences may need external locking. |
| 24 | `CopyOnWriteArrayList` is a good default for write-heavy collections. | **False** | Writes copy the underlying array. |
| 25 | Java 21 added sequenced collection interfaces. | **True** | They standardize encounter-order operations. |
| 26 | `IdentityHashMap` uses ordinary `equals` key equality. | **False** | It uses reference identity. |
| 27 | `WeakHashMap` is a deterministic cache with a strict eviction policy. | **False** | Key reclamation depends on reachability/GC. |
| 28 | A comparator returning zero for distinct objects may cause a TreeSet to treat them as duplicates. | **True** | Ordering equivalence defines sorted-set uniqueness. |
| 29 | `HashMap.newHashMap(n)` is available in modern Java and sizes for expected mappings. | **True** | Added in Java 19. |
| 30 | Amortized O(1) append means every append takes identical constant time. | **False** | Some appends trigger array growth. |

## Compact summary

- Choose collections by required semantics: ordering, uniqueness, random access, priority ordering, range queries, or concurrency.
- `ArrayList` is usually the general-purpose list; `ArrayDeque` is the common queue/stack choice; `HashMap`/`HashSet` are expected O(1); `TreeMap`/`TreeSet` are sorted O(log n).
- Hash tables use hash codes to select buckets and equality to distinguish keys; collisions are normal. Keep map keys' equality/hash state stable.
- Learn `HashMap` bucket/resize/tree-bin mechanics, but label exact thresholds as implementation details—not Java API guarantees.
- Know encounter order, Java 21 sequenced collections, unmodifiable vs immutable vs fixed-size collections, and the limitations of fail-fast iterators.

## Official references

- [Collections and Streams — dev.java](https://dev.java/learn/api/collections-and-streams/)
- [HashMap API, Java SE 26](https://docs.oracle.com/en/java/javase/26/docs/api/java.base/java/util/HashMap.html)
- [Map API, Java SE 26](https://docs.oracle.com/en/java/javase/26/docs/api/java.base/java/util/Map.html)
- [ArrayList API, Java SE 26](https://docs.oracle.com/en/java/javase/26/docs/api/java.base/java/util/ArrayList.html)
- [ConcurrentHashMap API, Java SE 26](https://docs.oracle.com/en/java/javase/26/docs/api/java.base/java/util/concurrent/ConcurrentHashMap.html)
