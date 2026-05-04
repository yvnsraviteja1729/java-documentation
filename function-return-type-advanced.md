<p><a target="_blank" href="https://app.eraser.io/workspace/wLlQM1TSiUnY18xUqadc" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>

In Java, you cannot directly define a return type using a union operator (like `TypeA | TypeB`) as you do in TypeScript. Java is strictly typed, and a method must declare a single return type.

However, since the release of **Java 17**, we have a feature called **Sealed Classes**, which is the closest and most powerful alternative to TypeScript's union types.

Here are the three main ways to achieve "either/or" return types in Java:

---

### 1. The Modern Way: Sealed Classes (Java 17+)
Sealed classes allow you to define a restricted hierarchy. You create a parent interface or class and "seal" it so only specific subclasses can implement it. This allows the compiler to perform **exhaustive pattern matching**, just like TypeScript.

```java
public sealed interface OrderResult permits Success, Failure {}

public record Success(String orderId) implements OrderResult {}
public record Failure(String errorMessage) implements OrderResult {}

public OrderResult placeOrder(int amount) {
    if (amount > 0) {
        return new Success("ORD-123");
    } else {
        return new Failure("Invalid amount");
    }
}
```
**How to use it:**

```java
OrderResult result = placeOrder(50);

// Using switch expressions (Java 17/21+)
String message = switch (result) {
    case Success s -> "Order created: " + s.orderId();
    case Failure f -> "Error: " + f.errorMessage();
};
```
---

### 2. The Functional Way: The `Either` Pattern
If you use functional programming libraries like **Vavr**, or want to build a simple wrapper, you can use an `Either` type. This is a common pattern in Java to represent a value that can be one of two types (conventionally called `Left` for errors/alternatives and `Right` for the "right" value).

```java
public Either<String, Integer> parse(String input) {
    try {
        return Either.right(Integer.parseInt(input));
    } catch (NumberFormatException e) {
        return Either.left("Not a valid number");
    }
}
```
---

### 3. The Classic OOP Way: Common Interface
Before Java 17, the standard approach was to return a common interface that both "types" implement. While flexible, it doesn't give you the compile-time safety that ensures you've handled every possible type unless you use `instanceof` checks.

```java
public interface ApiResponse {}

public class UserResponse implements ApiResponse { ... }
public class ErrorResponse implements ApiResponse { ... }

public ApiResponse getCustomer() {
    if (found) return new UserResponse();
    return new ErrorResponse();
}
```
---

### Summary Table
| Feature | TypeScript Union | Java Sealed Class | Java `Either`  |
| ----- | ----- | ----- | ----- |
| **Syntax** | `TypeA | TypeB`  | `sealed interface Result`  | `Either<L, R>`  |
| **Safety** | Compile-time exhaustive | Compile-time exhaustive | Functional/Method-based |
| **Complexity** | Very Low | Low/Medium | Medium |
| **Best For** | Ad-hoc types | Business domain logic | Error handling/FP |
If you are working on a modern project (Java 17 or higher), **Sealed Classes** are the recommended approach because they provide the best balance of readability and type safety.





<!--- Eraser file: https://app.eraser.io/workspace/wLlQM1TSiUnY18xUqadc --->