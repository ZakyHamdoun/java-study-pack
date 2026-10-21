# Java Language Basics — New-Grad Interview Cheat Sheet

> Baseline: modern Java, with release-specific notes checked against Java 26 and Java 27 as of **20 October 2026**. Core language rules are stable; preview features are explicitly labeled.

## 1. Java, JDK, JVM, and the execution pipeline

- **Java language**: syntax and static type system defined by the Java Language Specification (JLS).
- **JVM**: loads/verifies/executes bytecode and manages runtime services such as memory and garbage collection. HotSpot is a JVM implementation, not the definition of Java.
- **JDK**: JVM plus compiler (`javac`), launcher (`java`), diagnostics (`jcmd`, `jstack`, `jfr`), libraries, and other development tools.
- **Java SE API**: standard library contract (`java.lang`, collections, concurrency, I/O, etc.).
- Typical path: `source.java` → `javac` → `.class` bytecode → class loading/linking/initialization → interpretation and/or JIT-compiled machine code. The JIT may optimize hot code using profiling; bytecode is not normally interpreted forever.
- Java is **statically typed**, but type inference exists (`var`, generic method inference, lambda target typing). `var` is not dynamic typing.
- **Portability** means class files target a specified class-file version and behavior is described by the Java specifications; native libraries, runtime configuration, and implementation differences can still affect portability.

```bash
javac --release 25 Main.java  # compile against the Java 25 language/API target
java Main
java -version
javac -version
```

`--release N` constrains language rules, generated class-file version, and public platform APIs available for target N. It is usually safer than separately managing `-source` and `-target`.

## 2. Java versions: the important current facts

- **Java 26 GA:** 17 March 2026.
- **Java 27 GA:** 15 September 2026. Java 27 is a feature release; Java 25 is the current LTS release in this timeline.
- Java 26 delivered HTTP/3 support in the `HttpClient` API and a fourth preview of primitive types in patterns / `instanceof` / `switch`, among other work.
- Java 27 continues previewing primitive patterns (fifth preview), structured concurrency (seventh preview), and lazy constants (third preview). Preview APIs/language features are not permanent compatibility promises and may change or be withdrawn.
- Java 27 made **G1 the default GC in all environments** and **compact object headers the default** in HotSpot. These are runtime implementation defaults, not syntax changes.
- To compile/run code using preview features, use the matching JDK and enable previews, e.g. `javac --enable-preview --release 27 Example.java` then `java --enable-preview Example`. Do not use preview code as if it were ordinary stable Java.

## 3. Primitive values, references, and literals

### The eight primitive types

| Type | Bits / range summary | Common trap |
|---|---|---|
| `boolean` | `true` / `false`; JVM representation is implementation detail | Not an integer type in Java source expressions |
| `byte` | 8-bit signed, -128..127 | Arithmetic promotes it to `int` |
| `short` | 16-bit signed | Arithmetic promotes it to `int` |
| `char` | 16-bit unsigned UTF-16 code unit, 0..65535 | A `char` is not necessarily an entire Unicode code point |
| `int` | 32-bit signed | Integer overflow wraps; no automatic exception |
| `long` | 64-bit signed | Add `L` to make a literal `long` when needed |
| `float` | 32-bit IEEE 754 binary floating point | Usually too imprecise for money |
| `double` | 64-bit IEEE 754 binary floating point | Decimal fractions are often not exact |

`byte`, `short`, and `char` undergo **binary numeric promotion** to at least `int` for most arithmetic. Assignment back to a narrower type generally needs an explicit cast (except supported constant-expression narrowing cases).

```java
byte b = 1;
// b = b + 1;        // compile error: expression is int
b += 1;              // compound assignment includes an implicit cast
int overflow = Integer.MAX_VALUE + 1; // Integer.MIN_VALUE; wraps
System.out.println(0.1 + 0.2);        // not exactly 0.3
BigDecimal price = new BigDecimal("0.10"); // use a decimal string for money
```

Use `BigDecimal` for decimal monetary arithmetic when decimal semantics matter; choose rounding mode and scale deliberately. `new BigDecimal(0.1)` captures the binary approximation of the `double` and is often not what you intend.

### Primitives vs reference types

- Primitive variables hold primitive values. Reference variables hold a reference to an object or `null`.
- Local variables do **not** get Java language default values; they must be definitely assigned before use. Instance fields and array elements do get default values (`0`, `false`, `\u0000`, or `null`).
- Java is **pass-by-value only**. When an object reference is passed, the copied value is the reference. A method can mutate the referenced object, but reassigning its parameter does not reassign the caller's variable.
- Arrays are objects with fixed length. Arrays are covariant (`String[]` is an `Object[]`), so an invalid store can fail at runtime with `ArrayStoreException`.

```java
static void change(StringBuilder x) { x.append("!"); }
static void reassign(StringBuilder x) { x = new StringBuilder("new"); }
var sb = new StringBuilder("hi");
change(sb);   // caller sees "hi!"
reassign(sb); // caller still references the original "hi!"
```

### Wrappers, boxing, and unboxing

- Wrapper types (`Integer`, `Long`, etc.) are objects; boxing converts a primitive to a wrapper and unboxing converts back.
- Unboxing `null` throws `NullPointerException`.
- Do not compare wrappers by `==` to compare numeric value. Wrapper caching makes some small boxed values reuse object identities; this is not a value-comparison contract. Use primitive values or `.equals`.
- Generics use reference types; `List<int>` is illegal, while `List<Integer>` is valid.

## 4. Variables, `final`, and `var`

- **Local variable**: declared in a method/block; must be definitely assigned before read.
- **Instance field**: one per object. **Static field**: one per class/class-loader identity.
- `final` local/field/parameter can be assigned once. A `final` reference cannot point to a different object after assignment, but the object itself may still be mutable.
- `static final` constants may be compile-time constants only when their declarations and initializers meet the language rules (for example, primitive/String constant expressions). Changing such a constant can have binary compatibility surprises because use sites may inline it.
- `var` infers a **local variable's static type** from an initializer. It cannot declare fields, method parameters, or a variable without a suitable initializer. It cannot infer `null` alone. It does not make a variable dynamically typed.

```java
var names = new ArrayList<String>(); // inferred ArrayList<String>
final List<String> xs = new ArrayList<>();
xs.add("allowed"); // final reference; mutable list
// var mystery = null; // cannot infer a type
```

## 5. Equality, identity, and strings

- `==` compares primitive values for primitive operands; for references, it tests whether they are the same object reference (or both `null`).
- `.equals` expresses logical equality as defined by the class. It is not automatically value-based for every object; `Object.equals` defaults to identity.
- If you override `equals`, also override `hashCode`: equal objects must have equal hash codes. Unequal objects may collide.
- Keep hash-relevant state stable while an object is used as a hash-map key or hash-set member.
- `String` is immutable. Concatenation produces a result string; compiler/runtime optimizations may use `StringBuilder` or other strategies. Use `StringBuilder` for repeated mutable concatenation in loops when performance matters.
- String literals and interned strings may share instances. Do not rely on `==` for string content.

```java
String a = new String("java");
String b = new String("java");
System.out.println(a == b);      // false
System.out.println(a.equals(b)); // true
```

## 6. Operators and control flow

- `&&` and `||` short-circuit; `&` and `|` can operate on booleans without short-circuiting (and are also bitwise operators for integral operands).
- `==` on references is not a replacement for `.equals`. `instanceof` is false for `null`.
- `switch` statements and switch expressions are both available. Modern switch supports arrow rules, multiple labels, and `yield` in a block rule that returns a value.
- Fall-through occurs in traditional colon-style `case` groups unless terminated; arrow rules do not fall through.
- `break` exits the nearest loop/switch; `continue` advances a loop; labeled `break`/`continue` can target an enclosing statement.
- `finally` normally runs on both normal and abrupt exits, but process termination or VM failure can prevent it; avoid returning/throwing from `finally` because it can suppress the original result/exception.
- `assert` is typically disabled unless enabled with `-ea`; never use it for public input validation or essential side effects.

```java
String label = switch (status) {
    case 200, 201 -> "success";
    case 400 -> "client error";
    default -> {
        log(status);
        yield "other";
    }
};
```

**Java 27 preview note:** primitive types in patterns, `instanceof`, and `switch` are still a preview feature in Java 27. Do not present primitive pattern matching as a generally released feature. Object pattern matching and pattern matching for `switch` reached final status in earlier releases; primitive patterns are distinct.

## 7. Methods, overloads, and dispatch

- A method signature for overload resolution consists mainly of method name and parameter types; return type alone cannot overload a method.
- **Overloading** is compile-time selection among methods with the same name and different parameter lists.
- **Overriding** is a subtype providing an instance-method implementation for a compatible superclass/interface method; virtual dispatch chooses the implementation based on the runtime object type.
- `static` methods are hidden rather than overridden. Fields are also hidden, not dynamically overridden.
- A covariant return type lets an override return a subtype of the declared return type. An override cannot reduce visibility or broaden checked exceptions beyond what the inherited contract allows.
- Varargs are arrays at the call boundary; overload resolution and boxing/widening can make overloaded calls surprising.
- Java does not support user-defined operator overloading.

## 8. Initialization order

For a newly constructed class instance, a useful simplified sequence is:

1. Static field initializers and static initializer blocks of classes run once as each class is initialized, superclass before subclass.
2. Memory for the object is allocated and fields get default values.
3. The superclass constructor chain runs.
4. Instance field initializers and instance initializer blocks run in textual order for each class as constructor execution reaches it.
5. The constructor body runs.

`this(...)` constructor chaining must be the first constructor invocation under traditional Java syntax rules (newer language evolution may add special constructor-body forms; follow the target release's JLS). Be cautious when constructors call overridable methods: the subclass fields may not yet be initialized.

## 9. Exceptions and resource safety

- **Checked exceptions** must be caught or declared, excluding `RuntimeException` subclasses and `Error` subclasses from the checked category.
- **Unchecked exceptions** (`RuntimeException`) usually indicate programming or contract failures; `Error` usually indicates serious VM/environmental problems and normally is not caught as routine control flow.
- `throw` throws an exception object; `throws` declares possible exceptions in a method signature.
- Catch specific exceptions before their supertypes. Do not swallow an exception silently.
- **Try-with-resources** closes `AutoCloseable` resources in reverse declaration order and preserves the primary exception while recording close failures as suppressed exceptions.

```java
try (var in = Files.newBufferedReader(path)) {
    return in.readLine();
} // reader closes automatically
```

## 10. Packages, access, and API compatibility

- `public`: accessible wherever the enclosing type/module exports it. `protected`: package access plus subclass access rules. Package-private (no modifier): same package. `private`: declaring top-level type's permitted nest access.
- A Java package is a naming/access boundary, not by itself a security sandbox. The module system adds explicit readability and exports.
- `import` resolves names at compile time; it does not copy code or load a class immediately.
- `null` has no type you can declare directly; dereferencing it throws `NullPointerException`.
- Binary/source/behavioral compatibility are different. Code that compiles against one API version can still change behavior or fail when linked/executed against another runtime.

## 11. Common interview traps (True / False)

Answers and the reason are included so this can be used for self-testing. Cover the answer column while revising.

| # | Statement | Answer | Why |
|---:|---|:---:|---|
| 1 | Java passes object references by reference. | **False** | Java passes the reference value by value. |
| 2 | A local `int` is initialized to zero automatically. | **False** | Local variables require definite assignment. |
| 3 | An instance field of type `int` defaults to zero. | **True** | Fields and array elements receive default values. |
| 4 | `final List<String> x` makes the list immutable. | **False** | The reference cannot be reassigned; the list may be mutable. |
| 5 | `String` objects are mutable. | **False** | `String` is immutable. |
| 6 | `==` compares string contents. | **False** | For references it compares identity. |
| 7 | Equal objects must have equal hash codes. | **True** | This is required by the `equals`/`hashCode` contract. |
| 8 | Unequal objects must have different hash codes. | **False** | Hash collisions are legal. |
| 9 | `byte b = 1; b = b + 1;` compiles. | **False** | `b + 1` is an `int` expression. |
| 10 | Integer overflow normally throws an exception. | **False** | Integral overflow wraps around. |
| 11 | `0.1 + 0.2 == 0.3` is guaranteed true. | **False** | Binary floating-point cannot represent many decimal fractions exactly. |
| 12 | Unboxing a `null Integer` succeeds as zero. | **False** | It throws `NullPointerException`. |
| 13 | Arrays are covariant. | **True** | `String[]` is an `Object[]`; incompatible stores fail at runtime. |
| 14 | `List<String>` can be assigned to `List<Object>`. | **False** | Generic parameterizations are invariant. |
| 15 | A method can be overloaded using only its return type. | **False** | Return type alone does not distinguish overloads. |
| 16 | An overridden instance method is dispatched based on runtime class. | **True** | Dynamic dispatch selects the implementation. |
| 17 | A static method is overridden polymorphically. | **False** | Static methods are hidden. |
| 18 | `finally` is absolutely guaranteed to run under every circumstance. | **False** | Process/VM termination and catastrophic failures are exceptions. |
| 19 | Try-with-resources closes resources in reverse order. | **True** | Last declared resource closes first. |
| 20 | `var` makes Java dynamically typed. | **False** | Its type is inferred once at compile time. |
| 21 | An enum can have fields, constructors, and methods. | **True** | Enums are specialized classes with restricted instance creation. |
| 22 | `char` always represents one user-perceived character. | **False** | It is one UTF-16 code unit; code points and grapheme clusters may need multiple units. |
| 23 | `&&` evaluates the right operand even when the left is false. | **False** | It short-circuits. |
| 24 | `switch` arrow rules fall through to the next case. | **False** | Arrow rules do not fall through. |
| 25 | Java 27 primitive patterns are a finalized, non-preview feature. | **False** | They are preview in Java 27. |

## Compact summary

- Java source is statically typed and compiled to bytecode; runtime execution may interpret and JIT-compile it.
- Java is pass-by-value, including when the value is an object reference.
- Distinguish primitive value equality, reference identity (`==`), and logical equality (`equals`). Keep the `equals`/`hashCode` contract.
- Know primitive promotion, overflow, floating-point precision, wrapper unboxing, array covariance, and generic invariance.
- Understand overload vs override, fields/static method hiding, initialization order, exception categories, and try-with-resources.
- Java 27 is GA; use preview flags only for features explicitly marked preview. G1 and compact object headers are Java 27 defaults in HotSpot.

## Official references

- [Java Language Basics — dev.java](https://dev.java/learn/language/constructs/basics/)
- [Java 26 release notes](https://www.oracle.com/java/technologies/javase/26-relnote-issues.html)
- [Java 27 release notes](https://www.oracle.com/java/technologies/javase/27all-relnotes.html)
- [JLS, Java SE 27](https://docs.oracle.com/javase/specs/jls/se27/html/index.html)
