# Object-Oriented Programming — New-Grad Interview Cheat Sheet

## 1. The four OOP principles

### Encapsulation

Bundle state and behavior and protect invariants behind a small public API. It is not just “make fields private”; it means callers cannot put the object into an invalid state through the supported API.

```java
public final class BankAccount {
    private long cents;

    public BankAccount(long openingCents) {
        if (openingCents < 0) throw new IllegalArgumentException();
        this.cents = openingCents;
    }

    public synchronized void deposit(long amount) {
        if (amount <= 0) throw new IllegalArgumentException("amount must be positive");
        cents = Math.addExact(cents, amount);
    }

    public synchronized long balanceCents() { return cents; }
}
```

- Prefer the least permissive visibility that satisfies the contract.
- Avoid exposing mutable internal collections directly; return an unmodifiable view or defensive copy as appropriate.
- Immutability is a strong encapsulation strategy, but not every encapsulated object must be immutable.

### Abstraction

Expose what a client needs to know and hide implementation details. Interfaces, abstract classes, and small domain APIs are common tools. Abstraction is about the contract; encapsulation is about controlling state/implementation boundaries.

### Inheritance

A subtype reuses/extends behavior and should be substitutable for its supertype. Use it for a genuine **is-a** relationship, not merely code reuse. Prefer composition when behavior can be assembled from collaborators without inheriting fragile implementation details.

### Polymorphism

Code written against a supertype can work with different concrete implementations. Instance-method overriding is selected at runtime; overload resolution is selected at compile time.

```java
interface PriceRule { BigDecimal priceFor(Order order); }
final class RegularPrice implements PriceRule { /* ... */ }
final class DiscountPrice implements PriceRule { /* ... */ }

PriceRule rule = chooseRule(order); // runtime implementation varies
BigDecimal price = rule.priceFor(order);
```

## 2. Classes, constructors, and `Object`

- An object has identity, state, and behavior. A class defines a type and a construction/behavior model.
- Constructors are not inherited or overridden. A constructor may chain to another constructor with `this(...)` or to a superclass constructor with `super(...)`.
- If no constructor is declared, the compiler may provide a no-argument constructor only when language conditions allow it; it does not add one when you declared another constructor.
- Every class except `Object` has a superclass. Java supports single class inheritance but multiple interface inheritance.
- Key `Object` methods: `equals`, `hashCode`, `toString`, `getClass`, `clone` (often best avoided), `wait`, `notify`, `notifyAll`.
- Avoid invoking overridable instance methods from constructors; subclass initialization may be incomplete.

## 3. Abstract class vs interface

| Capability | Abstract class | Interface |
|---|---|---|
| Type relationship | A class extends at most one class | A class can implement many interfaces |
| State | Can have instance fields, including mutable state | Fields are implicitly `public static final` constants |
| Constructors | Yes | No instance constructors |
| Concrete methods | Yes | Yes: `default`, `static`, and `private` methods |
| Abstract methods | Yes | Yes, for ordinary instance method declarations without bodies |
| Visibility | Class members can use normal access modifiers | Abstract interface methods are implicitly `public`; private helper methods are allowed |
| Best use | Shared base state/invariants or skeletal implementation | Capability/contract that unrelated types can implement |

Abstract classes cannot be instantiated. An abstract class may have **zero** abstract methods (for example, to forbid direct instantiation while providing shared behavior). A concrete subclass must implement inherited abstract methods unless it remains abstract.

```java
abstract class Shape {
    abstract double area();
    public void printArea() { System.out.println(area()); }
}

interface Describable {
    String description();
    default String shortDescription() { return description().strip(); }
    static boolean isBlank(String value) { return value == null || value.isBlank(); }
    private static String normalize(String value) { return value.strip(); }
}
```

The choice is not “interfaces have no implementation” (false in modern Java) or “abstract classes are always better for reuse.” Interface defaults support interface evolution and reusable behavior, but should not become a substitute for well-designed object state.

## 4. Interface method rules and default-method conflicts

- An interface method declared without `default`, `static`, or `private` and with no body is implicitly `public abstract` (with exceptions for signatures matching certain `Object` methods).
- `default` methods are inherited instance methods with a body. Implementing classes can override them.
- `static` interface methods belong to the interface and are called with the interface name; they are not inherited as instance methods.
- `private` interface methods (including private static helpers) are implementation helpers used by other interface methods; they are not visible to implementing classes.
- **Conflict rule:** a class implementation wins over an interface default. A more-specific subinterface default wins over a less-specific one. If two unrelated interfaces provide conflicting defaults, the implementing class must resolve the conflict explicitly.

```java
interface A { default String name() { return "A"; } }
interface B { default String name() { return "B"; } }
final class C implements A, B {
    @Override public String name() { return A.super.name(); }
}
```

If a superclass already implements a method with the signature, the class implementation wins (the “class wins” rule). Interface fields are constants; they do not provide per-instance state.

## 5. Overloading, overriding, hiding, and dispatch

| Concept | Selected when? | Typical discriminator |
|---|---|---|
| Overloading | Compile time | Same method name, different parameter list |
| Overriding | Runtime for instance invocation | Subclass implementation of inherited instance method |
| Static hiding | Compile time based on qualifying type | Static method with matching signature in subclass |
| Field hiding | Compile time based on reference's declared type | Field declared again in subclass |

- An override cannot have weaker accessibility, a broader checked-exception contract, or an incompatible return type.
- Covariant returns permit a narrower reference return type.
- `@Override` lets the compiler catch mistakes and should normally be used when overriding.
- Private methods are not overridden in the usual polymorphic sense; final methods cannot be overridden; final classes cannot be subclassed.
- Overloads are resolved from compile-time argument types and applicable conversions; they do not choose an overload based on an argument's runtime class. Once an instance signature is selected, overriding still chooses implementation dynamically.

```java
class Parent {
    String value() { return "parent"; }
    static String kind() { return "parent"; }
}
class Child extends Parent {
    @Override String value() { return "child"; }
    static String kind() { return "child"; }
}
Parent p = new Child();
p.value(); // "child"
p.kind();  // "parent"; static selection uses the type Parent
```

## 6. `final`, immutable classes, records, and enums

- `final class`: cannot be subclassed. `final method`: cannot be overridden. `final variable`: assigned at most once.
- A truly immutable class typically has private final fields, no mutators, no `this` escape during construction, and defensive copies for mutable inputs/outputs. `final` alone does not make referenced objects immutable.
- A **record** is a concise declaration for a transparent data carrier. Record components generate private final fields, accessors, a canonical constructor, and value-oriented `equals`/`hashCode`/`toString`. Records are implicitly final and extend `java.lang.Record`; they cannot extend arbitrary classes. Their fields are final, but a component that refers to a mutable object does not magically make that object immutable.
- An **enum** defines a fixed set of instances and may carry fields, methods, and constructors. Enum constructors are not public; each constant is initialized as a singleton enum instance.
- Use records for data-centric values, not automatically for every entity whose identity/lifecycle matters.

```java
record Point(int x, int y) {
    Point { // compact canonical constructor: validate/normalize components
        if (x < 0 || y < 0) throw new IllegalArgumentException();
    }
}
```

## 7. Sealed classes and interfaces

`sealed` restricts which types can directly extend/implement a type. Every direct permitted subtype must declare one of:

- `final`: cannot be further extended;
- `sealed`: continues to restrict its own subtypes;
- `non-sealed`: reopens unrestricted extension below that branch.

```java
public sealed interface Payment permits CardPayment, CashPayment {}
public record CardPayment(String lastFour) implements Payment {}
public final class CashPayment implements Payment {}
```

A permitted subtype must obey the JLS placement rules: in the same named module, or in the same package when the types are in an unnamed module. The `permits` list can sometimes be inferred when direct subtypes are declared suitably in the same compilation unit. Sealed types are useful when the domain has a deliberately closed set of alternatives, especially with exhaustive pattern matching.

Do not confuse:

- `final`: closes inheritance completely;
- `sealed`: permits a chosen set of direct subtypes;
- `non-sealed`: removes the sealed restriction for that branch;
- package-private: limits visibility, not the set of inheritance relationships in exactly the same way.

## 8. Composition and dependency inversion

Composition means an object delegates work to other objects it holds. It helps isolate change and allows behavior to be configured/tested independently.

```java
final class CheckoutService {
    private final PaymentGateway gateway;
    CheckoutService(PaymentGateway gateway) { this.gateway = gateway; }
    Receipt checkout(Order order) { return gateway.charge(order.total()); }
}
```

Interview design heuristics:

- Prefer programming to interfaces, but do not create an interface for every class without a reason.
- Separate policy from implementation details.
- Favor immutable value objects and explicit dependencies.
- Keep invariants inside the object that owns the relevant state.
- Use inheritance for substitutable types; use composition for “has-a” and pluggable behavior.

### SOLID at interview depth

- **S — Single Responsibility:** a module should have a focused reason to change.
- **O — Open/Closed:** extend behavior without repeatedly rewriting stable code; use this as a design goal, not a ban on edits.
- **L — Liskov Substitution:** a subtype must honor the behavioral contract of its supertype, not just compile.
- **I — Interface Segregation:** clients should not depend on methods they do not need.
- **D — Dependency Inversion:** high-level policy should depend on abstractions rather than concrete low-level details; dependencies can be injected.

## 9. Common interview traps (True / False)

| # | Statement | Answer | Why |
|---:|---|:---:|---|
| 1 | Encapsulation is equivalent to declaring every field `private`. | **False** | Privacy helps, but encapsulation is about protecting invariants and controlling access. |
| 2 | An abstract class must contain an abstract method. | **False** | It may have none. |
| 3 | An abstract class can have a constructor. | **True** | The constructor initializes its part of subclass instances. |
| 4 | A Java class can extend two classes. | **False** | Java has single class inheritance. |
| 5 | A class can implement multiple interfaces. | **True** | Multiple interface inheritance is supported. |
| 6 | Modern interfaces can contain implemented instance methods. | **True** | They can declare `default` methods. |
| 7 | Interface fields are per-instance fields. | **False** | They are implicitly `public static final`. |
| 8 | Interface static methods are inherited as instance methods. | **False** | They are called on the declaring interface. |
| 9 | A private interface method is an API method for implementing classes. | **False** | It is an internal helper. |
| 10 | A class method implementation wins over an interface default of the same signature. | **True** | The class-wins rule applies. |
| 11 | Two unrelated interfaces with conflicting defaults always compile without a class override. | **False** | The conflict must be resolved. |
| 12 | Return type alone can distinguish overloads. | **False** | It is not enough. |
| 13 | Static methods participate in runtime overriding. | **False** | They are hidden and selected using compile-time type. |
| 14 | Fields are dynamically dispatched like overridden instance methods. | **False** | Field selection follows the declared reference type. |
| 15 | `@Override` can help detect a misspelled/mismatched override. | **True** | The compiler checks that overriding occurs. |
| 16 | A `final` reference guarantees the referenced object is immutable. | **False** | The reference is final; the object may be mutable. |
| 17 | Java records can extend any application-defined superclass. | **False** | They implicitly extend `Record`. |
| 18 | A record containing a `List` component guarantees that list cannot be mutated. | **False** | The component reference is final, but the list may be mutable. |
| 19 | Enum constants can have associated state and behavior. | **True** | Enums are specialized classes. |
| 20 | A sealed type is simply a final type with another name. | **False** | It permits an explicit set of direct subtypes. |
| 21 | Every direct subtype of a sealed class must be `final`, `sealed`, or `non-sealed`. | **True** | This is the sealed hierarchy rule. |
| 22 | `non-sealed` allows a branch to be extended without the sealed parent's restriction. | **True** | That is its purpose. |
| 23 | A subtype is valid if it compiles, even when it violates the superclass's behavioral promises. | **False** | Substitutability is behavioral (Liskov). |
| 24 | Composition is often safer than inheritance for reusing implementation. | **True** | It reduces coupling to superclass behavior. |
| 25 | Calling an overridable method from a constructor is always safe. | **False** | A subclass may not be initialized yet. |

## Compact summary

- Four principles: encapsulation protects invariants, abstraction simplifies the contract, inheritance models substitutable “is-a” relationships, and polymorphism allows implementations to vary behind a stable type.
- Abstract classes can share state and implementation; interfaces define capabilities and may now have default/static/private methods.
- Know the “class wins” and “most-specific interface wins” rules, and explicitly resolve unrelated default conflicts.
- Overload resolution is compile-time; overridden instance implementation selection is runtime. Static methods and fields are hidden, not overridden.
- Use records for value-like data; enums for fixed choices; sealed hierarchies for closed alternatives; composition for interchangeable behavior.

## Official references

- [Abstract methods and classes — dev.java](https://dev.java/learn/inheritance/abstract-classes/)
- [Java Language Specification, Java SE 27](https://docs.oracle.com/javase/specs/jls/se27/html/index.html)
- [Java 27 release notes](https://www.oracle.com/java/technologies/javase/27all-relnotes.html)
