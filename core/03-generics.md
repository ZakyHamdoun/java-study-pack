# Generics — New-Grad Interview Cheat Sheet

## 1. Why generics exist

Generics let classes, interfaces, and methods parameterize their types. They provide compile-time type safety, reduce explicit casts, and support reusable algorithms.

```java
List<String> names = new ArrayList<>();
names.add("Ada");
String first = names.get(0); // no cast
```

Without generics, collections commonly held `Object`, so callers had to cast and could discover type errors only at runtime. Generics do **not** mean the JVM normally creates a wholly separate class for each type argument; Java implements them largely through type erasure.

## 2. Type parameters and generic methods

```java
final class Box<T> {
    private T value;
    Box(T value) { this.value = value; }
    T get() { return value; }
    void set(T value) { this.value = value; }
}

static <T> T first(List<T> values) {
    return values.get(0);
}

static <T extends Comparable<? super T>> T max(T a, T b) {
    return a.compareTo(b) >= 0 ? a : b;
}
```

- Type parameter declaration: `<T>`, `<K, V>`, etc. Names are conventions, not keywords (`T`, `E`, `K`, `V`, `R`).
- A generic method declares its own parameters before the return type, e.g. `<T> T first(...)`; it may be static because the method's type variables are independent of an instance's type parameters.
- Compiler inference often fills in type arguments. You may supply them explicitly when clarity or inference requires it: `Generics.<String>first(names)`.
- Bounds restrict allowable types. `T extends Number` means `T` is `Number` or a subtype; `extends` is used for both class and interface upper bounds.
- With multiple bounds, a class bound (if present) must appear first: `<T extends Number & Comparable<T>>`.
- `Comparable<? super T>` is common in reusable APIs because a `T` can be compared through `Comparable<T>` or a compatible supertype.

## 3. Invariance: the rule behind many interview questions

Even if `Integer` is a subtype of `Number`, `List<Integer>` is **not** a subtype of `List<Number>`. If it were, someone could add a `Double` to a `List<Integer>` through a `List<Number>` reference.

```java
List<Integer> ints = new ArrayList<>();
// List<Number> numbers = ints; // does not compile
```

Arrays differ: `Integer[]` can be assigned to `Number[]` because arrays are covariant, but an invalid store fails at runtime. Generic invariance prevents this category of write from compiling.

## 4. Wildcards and PECS

A wildcard `?` means an unknown type argument. The two useful bounds are:

- `? extends T`: unknown subtype of `T` (upper-bounded wildcard). You can safely read elements as `T`, but generally cannot add a non-null `T` because the actual list could be `List<SomeNarrowSubtype>`.
- `? super T`: unknown supertype of `T` (lower-bounded wildcard). You can safely add `T` values, but a read is only known to be `Object`.
- `?`: use when the specific element type is irrelevant. You can read as `Object`; you cannot add a non-null value (except `null`) because the element type is unknown.

**PECS: Producer Extends, Consumer Super.** If a parameter only produces values for your code to read, consider `? extends T`; if it consumes `T` values supplied by your code, consider `? super T`.

```java
static double sum(List<? extends Number> numbers) {
    double total = 0;
    for (Number n : numbers) total += n.doubleValue();
    return total;
}

static void copy(List<? extends Number> source,
                 List<? super Number> destination) {
    for (Number n : source) destination.add(n);
}
```

`List<? extends Number>` does not mean “a list of any mix of Number subtypes”; it means “a list whose one unknown element type is some subtype of Number.”

### Wildcard capture

Some operations are difficult because a wildcard hides a type variable. A helper generic method can capture it as a named type parameter:

```java
static void swap(List<?> list, int i, int j) { swapCaptured(list, i, j); }
private static <T> void swapCaptured(List<T> list, int i, int j) {
    T old = list.get(i);
    list.set(i, list.get(j));
    list.set(j, old);
}
```

## 5. Type erasure, bridges, and reifiable types

The compiler erases generic type parameters for bytecode/runtime compatibility:

1. An unbounded type parameter is generally erased to `Object`; a bounded one is erased to its first bound.
2. The compiler inserts casts where required at use sites.
3. It may generate **bridge methods** so an overriding method continues to work after erasure.

Consequences:

- At ordinary runtime, `new ArrayList<String>().getClass() == new ArrayList<Integer>().getClass()` is true.
- You cannot normally test `obj instanceof List<String>` because the runtime does not carry `String` as the generic argument. `obj instanceof List<?>` is allowed because the unbounded wildcard type is reifiable.
- `T.class` and `new T()` are not permitted in ordinary generic code; a `Class<T>` token or factory can be passed when runtime type/construction information is needed.
- Static fields cannot use a class's type parameter as a per-instantiation field type: there is one static member per runtime class, not one per `Box<String>` vs `Box<Integer>`.

```java
List<String> a = new ArrayList<>();
List<Integer> b = new ArrayList<>();
System.out.println(a.getClass() == b.getClass()); // true
```

## 6. Important restrictions

- No primitive type arguments: use `List<Integer>`, not `List<int>`.
- Cannot create an array of a concrete parameterized type like `new List<String>[10]`; arrays know their component type at runtime, while generic type arguments are erased.
- Cannot normally use `new T()`, `T.class`, or `instanceof T`.
- Cannot overload methods whose signatures become the same after erasure.
- Catch clauses cannot catch a type variable such as `catch (T e)` because the actual runtime type may be erased.
- Generic classes cannot directly extend `Throwable` to create parameterized exception classes.
- Raw types (e.g. `List` instead of `List<String>`) exist for legacy compatibility but bypass checks and can cause heap pollution. Prefer parameterized types and avoid raw APIs except at unavoidable legacy boundaries.

## 7. Heap pollution, unchecked operations, and varargs

**Heap pollution** occurs when a variable of a parameterized type refers to an object that is not of that parameterized type. It often stems from raw types, unchecked casts, or unsafe generic varargs.

```java
List<String> strings = new ArrayList<>();
List raw = strings;       // raw conversion; warning
raw.add(42);              // unchecked/unsafe
// String s = strings.get(0); // ClassCastException at this use
```

- Treat unchecked warnings as design/compatibility signals, not harmless noise.
- `@SuppressWarnings("unchecked")` should be narrow and accompanied by a proof/reason in a comment.
- Generic varargs may expose a non-reifiable array (`T...`). `@SafeVarargs` is a promise from the author that the method/constructor does not perform unsafe operations with that varargs array. It is not a runtime safety mechanism.
- `Collections.checkedList` can add runtime checks at a legacy boundary, but it does not erase the root design problem.

## 8. Designing generic APIs well

- Use type parameters when the relationship among types matters; use `?` when callers should not depend on the exact type argument.
- Prefer bounded wildcards for flexible input/output APIs rather than over-constraining callers.
- Return the most useful abstraction instead of exposing implementation types for no reason.
- Avoid over-generalizing tiny private methods; generics should make a relationship safer/clearer, not merely add angle brackets.
- For comparisons: `Comparable<T>` defines a type's natural ordering; `Comparator<T>` defines an external/alternative ordering. Comparators should ideally be consistent with `equals` when used in sorted sets/maps, or users can observe surprising duplicate behavior.

## 9. Common interview traps (True / False)

| # | Statement | Answer | Why |
|---:|---|:---:|---|
| 1 | `List<Integer>` is a subtype of `List<Number>`. | **False** | Parameterized types are invariant. |
| 2 | `Integer[]` is a subtype of `Number[]`. | **True** | Arrays are covariant (with runtime store checks). |
| 3 | `List<? extends Number>` is safe to read as `Number`. | **True** | Every possible element is some Number subtype. |
| 4 | You can always add a `Double` to `List<? extends Number>`. | **False** | The actual element type may be `Integer`. |
| 5 | You can add an `Integer` to `List<? super Integer>`. | **True** | Any permitted element type is Integer or a supertype. |
| 6 | A value read from `List<? super Integer>` is guaranteed to be an `Integer`. | **False** | It is only guaranteed to be an `Object`. |
| 7 | PECS means Producer Extends, Consumer Super. | **True** | This is the practical wildcard mnemonic. |
| 8 | A wildcard list's element type is always Object. | **False** | Its actual type is unknown; reads are exposed as Object when using `List<?>`. |
| 9 | A generic method can declare its own type parameter. | **True** | Example: `<T> T identity(T value)`. |
| 10 | Type parameters can be primitive types. | **False** | Use wrapper/reference types. |
| 11 | `new List<String>[10]` is allowed. | **False** | Generic array creation is forbidden for non-reifiable component types. |
| 12 | `new T()` is normally allowed in a generic class. | **False** | Erasure means there is no known runtime constructor for T. |
| 13 | `List<String>` and `List<Integer>` generally have different runtime classes. | **False** | Both are usually the same raw class after erasure. |
| 14 | `instanceof List<?>` is allowed. | **True** | It is reifiable. |
| 15 | `instanceof List<String>` is generally allowed. | **False** | The generic argument is not reified. |
| 16 | Erasure can require bridge methods. | **True** | Bridges preserve overriding after type erasure. |
| 17 | A raw type is as type-safe as a parameterized type. | **False** | It suppresses generic checks at the boundary. |
| 18 | Erasure replaces all bounded parameters with Object. | **False** | A bounded parameter erases to its first bound. |
| 19 | An unchecked cast is guaranteed to fail immediately if wrong. | **False** | Due to erasure, failure may occur only at a later use-site cast. |
| 20 | Static members have separate copies for `Box<String>` and `Box<Integer>`. | **False** | There is one runtime class after erasure. |
| 21 | Generic exception classes extending Throwable are allowed. | **False** | Generic subclasses of Throwable are forbidden. |
| 22 | `? super T` is often useful for consumers of T. | **True** | It accepts T and its subtypes safely. |
| 23 | `? extends T` always means the collection is immutable. | **False** | It restricts writes through that reference; another alias may mutate it. |
| 24 | `@SafeVarargs` performs runtime type checks. | **False** | It is a compiler-facing safety assertion, not a runtime mechanism. |
| 25 | The `Comparable<T>` ordering should be considered when using objects in TreeSet/TreeMap. | **True** | Ordering equivalence can affect set/map uniqueness. |

## Compact summary

- Generics move type errors to compile time and improve API relationships, but normal Java generics use **type erasure**.
- Parameterized types are invariant; use bounded wildcards for variance: **PECS** (producer `extends`, consumer `super`).
- Upper bounds are safe to read as the bound but not to write arbitrary values; lower bounds are safe to write the lower-bound type but reads are only `Object`.
- Know erasure, bridge methods, reifiable types, and restrictions around arrays, `T.class`, `new T()`, and generic exceptions.
- Avoid raw types and unchecked operations; use narrow, justified suppressions only when you can prove safety.

## Official references

- [Generics — dev.java](https://dev.java/learn/generics/)
- [Type erasure — dev.java](https://dev.java/learn/language/fp/generics/type-erasure/)
- [Java Language Specification, Java SE 27](https://docs.oracle.com/javase/specs/jls/se27/html/index.html)
