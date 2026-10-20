# Lambdas and Streams — New-Grad Interview Cheat Sheet

## 1. Functional interfaces and lambdas

A **functional interface** has one abstract method (SAM: single abstract method), ignoring methods that correspond to public `Object` methods and noting that default/static/private methods do not add another abstract method. `@FunctionalInterface` asks the compiler to verify the intent.

```java
@FunctionalInterface
interface IntTransformer { int apply(int value); }
IntTransformer twice = x -> x * 2;
```

Common built-in interfaces in `java.util.function`:

| Interface | Abstract method shape | Typical use |
|---|---|---|
| `Predicate<T>` | `boolean test(T)` | filter/test |
| `Function<T,R>` | `R apply(T)` | transform |
| `Consumer<T>` | `void accept(T)` | side-effecting action |
| `Supplier<T>` | `T get()` | create/provide value |
| `UnaryOperator<T>` | `T apply(T)` | T → T transform |
| `BinaryOperator<T>` | `T apply(T,T)` | combine two same-type values |
| `BiFunction<T,U,R>` | `R apply(T,U)` | two inputs to result |
| `BiPredicate<T,U>` | `boolean test(T,U)` | two-input test |
| `BiConsumer<T,U>` | `void accept(T,U)` | two-input action |

Primitive-specialized interfaces such as `IntPredicate`, `IntUnaryOperator`, `IntFunction<R>`, and `ToIntFunction<T>` can avoid boxing/unboxing.

### Lambda syntax and target typing

```java
Runnable r = () -> System.out.println("run");
Comparator<String> byLength = (a, b) -> Integer.compare(a.length(), b.length());
Function<String, Integer> parse = Integer::parseInt;
```

A lambda's type comes from its **target functional interface**; lambdas are not standalone objects with a universally fixed type. Method references (`Type::staticMethod`, `obj::instanceMethod`, `Type::instanceMethod`, `Type::new`) are concise lambda forms when signatures match.

### Capturing local variables

A lambda may capture a local variable only if it is final or **effectively final**. That means its value/reference is assigned once and not reassigned later. An object referenced by the captured variable may still be mutable, but mutation creates normal thread-safety/design concerns.

```java
int limit = 10;
Predicate<Integer> small = n -> n < limit; // okay
// limit = 11; // would make the capture illegal
```

Checked exceptions are not automatically permitted by standard functional-interface methods such as `Function.apply`; wrap/translate them or define an interface whose method declares the checked exception.

## 2. Stream model: a pipeline, not a collection

A stream describes a computation over elements from a source; it is not itself a container that stores elements.

Typical pipeline:

`source → zero or more intermediate operations → terminal operation`

- **Intermediate operations**: `filter`, `map`, `flatMap`, `distinct`, `sorted`, `limit`, `skip`, `peek`. Usually lazy; they describe work and return another stream.
- **Terminal operations**: `toList`, `collect`, `reduce`, `count`, `forEach`, `min`, `max`, `findFirst`, `anyMatch`, etc. They trigger evaluation and consume the stream.
- A stream should be operated on only once. Reusing a consumed stream usually throws `IllegalStateException` (do not depend on implementation quirks).
- Laziness permits fusion: multiple intermediate operations can often be applied in one traversal rather than creating a full intermediate collection after every stage.
- Stateful operations like `sorted` and `distinct` may require buffering or substantial state; a stream pipeline is not automatically memory-free.

```java
List<String> result = orders.stream()
    .filter(Order::isPaid)
    .map(Order::customerEmail)
    .filter(Objects::nonNull)
    .distinct()
    .toList(); // terminal operation; unmodifiable list
```

## 3. The core operations

- `map`: one input → one output; changes shape/type.
- `flatMap`: one input → a stream of outputs, then flattens those outputs to one stream. Useful for nested collections or optional multi-results.
- `filter`: retain matching elements.
- `peek`: primarily for diagnostics; avoid relying on it for required side effects because short-circuiting/optimization can mean not every element is observed.
- `distinct`: removes duplicates using `equals`/`hashCode` for ordinary reference streams; can be stateful.
- `sorted`: uses natural order or comparator. May require buffering; stream is not sorted unless this or an ordered source provides the guarantee.
- `limit`/`skip`: size-limiting/slicing; can be expensive with some parallel ordered pipelines.
- `findFirst` respects encounter order if present; `findAny` may be more flexible for parallel work.
- `anyMatch`, `allMatch`, and `noneMatch` short-circuit where the answer is known.

## 4. `reduce` vs `collect`

- `reduce` combines elements into one value, such as sum/product/min, using an accumulator that should be associative for reliable parallel reduction.
- `collect` performs mutable reduction into a result container using a `Collector` (supplier, accumulator, combiner, finisher/characteristics). Use it for lists, maps, grouping, partitioning, and other structured results.
- Don't use `reduce` to mutate one shared `ArrayList`; that breaks reduction assumptions, especially in parallel.

```java
int total = numbers.stream().reduce(0, Integer::sum);
Map<String, List<Order>> byCustomer = orders.stream()
    .collect(Collectors.groupingBy(Order::customerId));
Map<String, BigDecimal> totals = orders.stream()
    .collect(Collectors.toMap(Order::customerId, Order::total,
        BigDecimal::add)); // third argument resolves duplicate keys
```

The accumulator should generally be associative (`(a op b) op c == a op (b op c)`) for parallel reduction. Floating-point addition is not mathematically associative in finite precision, so parallel sums may differ in low-order bits.

### High-value collectors

- `toList()`, `toSet()`, `toCollection(factory)`
- `joining(delimiter)` for strings
- `counting`, `summingInt`, `averagingInt`, `summarizingInt`
- `groupingBy(classifier)` with optional downstream collector
- `partitioningBy(predicate)` creates the true/false partitions
- `mapping`, `filtering`, `flatMapping`, `collectingAndThen`
- `toMap(keyMapper, valueMapper, mergeFunction)` when keys may collide

Collectors do not necessarily preserve a particular order unless their contract/target collection defines it.

## 5. Primitive streams and `Optional`

- `IntStream`, `LongStream`, and `DoubleStream` avoid much of the wrapper boxing in primitive pipelines and offer operations such as `sum`, `average`, and primitive ranges.
- `Optional<T>` represents a value that may be absent. Useful methods: `map`, `flatMap`, `filter`, `orElse`, `orElseGet`, `orElseThrow`, `ifPresent`.
- `orElse(value)` evaluates the fallback argument before invocation; `orElseGet(supplier)` computes it only if the optional is empty.
- Avoid calling `.get()` without proving presence; use a domain-appropriate fallback or exception.
- `Optional` is mostly intended as a return type, not as a blanket replacement for every nullable field/parameter.

## 6. Parallel streams: when and when not

`parallelStream()` or `stream().parallel()` can split work and combine results, often using the common `ForkJoinPool`. It is not a free speed switch.

Works best when:

- the data source splits efficiently;
- there is enough CPU-heavy independent work per element to justify scheduling/combining;
- operations are stateless or correctly associative;
- side effects and ordering do not impose heavy coordination.

Often disappointing or dangerous when:

- tasks block on I/O;
- there are few/cheap elements;
- the pipeline mutates shared state;
- the source splits poorly;
- required encounter order forces coordination;
- it competes with other work in the common pool.

```java
long count = values.parallelStream()
    .filter(this::expensivePurePredicate)
    .count();
```

Do not update an ordinary shared `ArrayList`/`HashMap` inside a parallel stream. Prefer collectors or thread-safe aggregation designed for the task. `forEach` may run actions in arbitrary order; `forEachOrdered` preserves encounter order when one exists but may reduce parallelism.

## 7. Side effects and implementation quality

- Prefer pure transformations and terminal aggregation over modifying external state inside `map`/`filter`.
- Streams are not always clearer than a loop. A loop is often better for complex control flow, early exits with detailed conditions, checked exceptions, or stateful algorithms.
- Beware accidental boxing (`Stream<Integer>` vs `IntStream`) in hot code.
- Avoid long pipelines that hide important business logic; name intermediate methods when that improves readability.
- A stream on a mutable source does not itself make source access thread-safe. Do not mutate a non-concurrent source during traversal unless its specific contract permits it.

## 8. Common interview traps (True / False)

| # | Statement | Answer | Why |
|---:|---|:---:|---|
| 1 | Every interface with one abstract method can be a lambda target. | **Mostly true** | The rule has formal details around `Object` methods, defaults, and intersection types; SAM is the right model. |
| 2 | A lambda can freely reassign a captured local variable. | **False** | Captured locals must be final/effectively final. |
| 3 | An effectively-final local's referenced object must be immutable. | **False** | The reference cannot be reassigned; the object may be mutable. |
| 4 | A method reference is an alternative syntax for a compatible lambda. | **True** | Its target functional interface determines compatibility. |
| 5 | Intermediate stream operations normally execute immediately. | **False** | They are generally lazy. |
| 6 | A terminal stream operation triggers pipeline evaluation. | **True** | It consumes the pipeline. |
| 7 | A stream can reliably be reused after `count()`. | **False** | Streams are single-use. |
| 8 | `map` and `flatMap` are equivalent. | **False** | `flatMap` flattens nested streams. |
| 9 | `peek` should hold essential business side effects. | **False** | It is primarily diagnostic and may not observe all elements in optimized/short-circuited pipelines. |
| 10 | `findAny` is allowed to return any matching element. | **True** | Especially useful in parallel pipelines. |
| 11 | `findFirst` and `findAny` have identical ordering guarantees. | **False** | `findFirst` respects encounter order; `findAny` need not. |
| 12 | `forEach` on a parallel stream guarantees encounter order. | **False** | Use `forEachOrdered` when order matters. |
| 13 | Every `collect(toList())` necessarily returns a mutable list. | **False** | Collector mutability characteristics differ; `Stream.toList()` specifically returns an unmodifiable list. |
| 14 | `Stream.toList()` returns an unmodifiable list. | **True** | Mutation methods are not supported. |
| 15 | A collector should be preferred over mutating a shared collection in a parallel stream. | **True** | Collectors support structured reduction. |
| 16 | `reduce` is ideal for building arbitrary mutable containers by side effect. | **False** | Use `collect` for mutable reduction. |
| 17 | Parallel reductions work correctly for all arbitrary non-associative operators. | **False** | The reduction needs suitable associative semantics. |
| 18 | Integer addition is associative within Java's fixed-width modular arithmetic. | **True** | Overflow wraps, so fixed-width integer addition is associative modulo 2^n. |
| 19 | Floating-point addition is exactly associative. | **False** | Rounding means grouping can change results. |
| 20 | `orElse` is lazy about evaluating its fallback argument. | **False** | The argument expression is evaluated before the call. |
| 21 | `orElseGet` can defer fallback computation. | **True** | It invokes the supplier only when empty. |
| 22 | `IntStream` can avoid some boxing compared with `Stream<Integer>`. | **True** | It uses primitive int elements. |
| 23 | Parallel streams are always faster than sequential streams. | **False** | Overheads and contention can dominate. |
| 24 | Blocking I/O is normally the ideal use case for the common-pool parallel stream. | **False** | Parallel streams are usually aimed at split-friendly data processing; virtual threads or explicit I/O concurrency may fit better. |
| 25 | A predicate passed to `filter` should normally be side-effect-free. | **True** | Side effects make semantics harder to reason about and can break parallel correctness. |
| 26 | `Collectors.toMap(k, v)` silently resolves duplicate keys by keeping the last one. | **False** | Without a merge function, duplicate keys throw `IllegalStateException`. |

## Compact summary

- Lambdas are typed against functional interfaces; know `Predicate`, `Function`, `Consumer`, `Supplier`, and their primitive specializations.
- Captured locals must be final/effectively final. An object they reference may still mutate, which does not make that mutation thread-safe.
- Streams are lazy pipelines, single-use, and executed by a terminal operation. Know `map` vs `flatMap`, `reduce` vs `collect`, ordering, short-circuiting, and collector merge functions.
- Parallel streams need efficient splitting, independent work, associative reductions, and controlled side effects. They are not automatically faster, and the common pool can become a contention point.

## Official references

- [Collections and Streams — dev.java](https://dev.java/learn/api/collections-and-streams/)
- [Stream map/filter/reduce — dev.java](https://dev.java/learn/api/streams/map-filter-reduce/)
- [Java SE 26 Stream API](https://docs.oracle.com/en/java/javase/26/docs/api/java.base/java/util/stream/Stream.html)
- [Java SE 26 Collectors API](https://docs.oracle.com/en/java/javase/26/docs/api/java.base/java/util/stream/Collectors.html)
