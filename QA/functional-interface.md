# Functional Interfaces in Java — Notes

## 1. What is a Functional Interface?

A **Functional Interface** is an interface that contains **exactly one abstract method** (SAM — *Single Abstract Method*).

- Introduced in **Java 8**.
- Can be implemented using **Lambda Expressions**, **Method References**, or **Anonymous Classes**.
- Forms the foundation of **Functional Programming** in Java.
- Can have any number of **default** and **static** methods.

---

## 2. `@FunctionalInterface` Annotation

```java
@FunctionalInterface
interface MyFunc {
    void execute();   // exactly one abstract method
}
```

- **Optional** but recommended.
- Compiler enforces the SAM rule — throws error if more than one abstract method exists.
- Prevents accidental modification of the interface.

---

## 3. Rules of a Functional Interface

✅ Must have **exactly one abstract method**.
✅ Can have **multiple default methods**.
✅ Can have **multiple static methods**.
✅ Can override **`Object` class methods** (`equals`, `hashCode`, `toString`) — they don't count as abstract.

```java
@FunctionalInterface
interface Demo {
    void run();                          // ✅ abstract
    default void log() { }               // ✅ default
    static void info() { }               // ✅ static
    boolean equals(Object obj);          // ✅ Object method (allowed)
}
```

---

## 4. Example — Before vs After Java 8

### 🔸 Before Java 8 (Anonymous Class)
```java
Runnable r = new Runnable() {
    @Override
    public void run() {
        System.out.println("Running...");
    }
};
```

### 🔸 After Java 8 (Lambda)
```java
Runnable r = () -> System.out.println("Running...");
```

---

## 5. Custom Functional Interface

```java
@FunctionalInterface
interface Calculator {
    int operate(int a, int b);
}

public class Test {
    public static void main(String[] args) {
        Calculator add = (a, b) -> a + b;
        Calculator mul = (a, b) -> a * b;

        System.out.println(add.operate(5, 3));  // 8
        System.out.println(mul.operate(5, 3));  // 15
    }
}
```

---

## 6. Built-in Functional Interfaces (`java.util.function`)

| Interface | Abstract Method | Description | Example |
|-----------|----------------|-------------|---------|
| **`Function<T, R>`** | `R apply(T t)` | Takes T, returns R | `s -> s.length()` |
| **`Predicate<T>`** | `boolean test(T t)` | Takes T, returns boolean | `n -> n > 0` |
| **`Consumer<T>`** | `void accept(T t)` | Takes T, returns nothing | `s -> sout(s)` |
| **`Supplier<T>`** | `T get()` | Takes nothing, returns T | `() -> "Hello"` |
| **`UnaryOperator<T>`** | `T apply(T t)` | Same input/output type | `n -> n * 2` |
| **`BinaryOperator<T>`** | `T apply(T,T)` | Two same-type inputs | `(a,b) -> a+b` |
| **`BiFunction<T,U,R>`** | `R apply(T,U)` | Two inputs, one result | `(a,b) -> a+b` |
| **`BiPredicate<T,U>`** | `boolean test(T,U)` | Two inputs → boolean | `(a,b) -> a.equals(b)` |
| **`BiConsumer<T,U>`** | `void accept(T,U)` | Two inputs, no return | `(k,v) -> sout(k+v)` |

---

## 7. Examples of Each Built-in Interface

### 🔹 Function
```java
Function<String, Integer> length = s -> s.length();
System.out.println(length.apply("Java"));   // 4
```

### 🔹 Predicate
```java
Predicate<Integer> isEven = n -> n % 2 == 0;
System.out.println(isEven.test(10));   // true
```

### 🔹 Consumer
```java
Consumer<String> print = s -> System.out.println(s);
print.accept("Hello");
```

### 🔹 Supplier
```java
Supplier<Double> random = () -> Math.random();
System.out.println(random.get());
```

### 🔹 UnaryOperator
```java
UnaryOperator<Integer> square = n -> n * n;
System.out.println(square.apply(5));   // 25
```

### 🔹 BinaryOperator
```java
BinaryOperator<Integer> sum = (a, b) -> a + b;
System.out.println(sum.apply(3, 4));   // 7
```

---

## 8. Commonly Used Pre-existing Functional Interfaces

These existed **before Java 8** but qualify as functional interfaces:

| Interface | Method |
|-----------|--------|
| `Runnable` | `void run()` |
| `Callable<V>` | `V call()` |
| `Comparator<T>` | `int compare(T, T)` |
| `Comparable<T>` | `int compareTo(T)` |
| `ActionListener` | `void actionPerformed(ActionEvent)` |

---

## 9. Default & Static Methods Example

```java
@FunctionalInterface
interface Vehicle {
    void start();                       // abstract

    default void stop() {
        System.out.println("Stopping...");
    }

    static void service() {
        System.out.println("Servicing...");
    }
}

public class Car {
    public static void main(String[] args) {
        Vehicle v = () -> System.out.println("Starting...");
        v.start();
        v.stop();
        Vehicle.service();
    }
}
```

---

## 10. Method References (Shortcut for Lambdas)

| Type | Syntax | Example |
|------|--------|---------|
| Static method | `Class::staticMethod` | `Integer::parseInt` |
| Instance method (specific obj) | `obj::method` | `System.out::println` |
| Instance method (any obj) | `Class::instanceMethod` | `String::toLowerCase` |
| Constructor reference | `Class::new` | `ArrayList::new` |

```java
Consumer<String> print = System.out::println;
print.accept("Hello");
```

---

## 11. Chaining Functional Interfaces

```java
Function<Integer, Integer> doubleIt = n -> n * 2;
Function<Integer, Integer> addTen = n -> n + 10;

System.out.println(doubleIt.andThen(addTen).apply(5)); // (5*2)+10 = 20
System.out.println(doubleIt.compose(addTen).apply(5)); // (5+10)*2 = 30
```

```java
Predicate<Integer> isPositive = n -> n > 0;
Predicate<Integer> isEven = n -> n % 2 == 0;

System.out.println(isPositive.and(isEven).test(4));  // true
System.out.println(isPositive.or(isEven).test(-3));  // false
System.out.println(isPositive.negate().test(-1));    // true
```

---

## 12. Advantages

✅ Enables **Lambda expressions** & **Method references**.
✅ Promotes **functional/declarative** programming.
✅ Cleaner, more **readable** code.
✅ Powers the **Stream API**.
✅ Reduces **boilerplate** (no anonymous class clutter).

---

## 13. Disadvantages

❌ Restricted to **one abstract method**.
❌ **Debugging** lambdas can be harder.
❌ Overuse can reduce code readability.

---

## 14. Best Practices

1. Use **`@FunctionalInterface`** to enforce contract.
2. Prefer **built-in interfaces** over custom ones when possible.
3. Use **method references** when they improve readability.
4. Keep lambdas **short** — extract to methods if logic is complex.
5. Combine with **Streams** for powerful data processing.

---

## Quick Recap

> **Functional Interface = Interface with exactly 1 abstract method** → enables **lambdas, method references, and Stream API** in Java 8+. Common ones: **`Function`, `Predicate`, `Consumer`, `Supplier`**.
