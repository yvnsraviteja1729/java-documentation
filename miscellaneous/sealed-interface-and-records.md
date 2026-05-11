<p><a target="_blank" href="https://app.eraser.io/workspace/mWDathqgEgnZNqRzdvv4" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>



Your snippet is a textbook example of **algebraic data types (ADTs)** in Java, built using two modern features: **sealed types** (Java 17+) and **records** (Java 16+). Let me break down what's happening, why it's powerful, and what the "old way" would look like.

---

## 1.
```java
public sealed interface AddToCartResult {
    record Success(Cart cart) implements AddToCartResult {}
    record OutOfStock(String sku, int available) implements AddToCartResult {}
    record ProductDiscontinued(String sku) implements AddToCartResult {}
}
```
Read this in plain English:

>  "An `AddToCartResult` is **exactly one of three things**: a `Success` carrying a `Cart`, an `OutOfStock` carrying a SKU and an available quantity, or a `ProductDiscontinued` carrying a SKU. Nothing else can ever be an `AddToCartResult`." 

That single sentence is enforced by the compiler. That is the whole point.

---

## 2. The Two Features Explained
### 2.1 `sealed interface` 
A `sealed` type **restricts which classes/interfaces can implement or extend it**.

- Normal `interface`  → anyone, anywhere, can implement it.
- `sealed interface`  → only a known, finite set of types can implement it.
There are three keywords for subtypes of a sealed type:

| Keyword | Meaning |
| ----- | ----- |
| `final`  | The subtype cannot be extended further. |
| `sealed`  | The subtype is itself sealed and must declare its own permitted subtypes. |
| `non-sealed`  | Opens the hierarchy back up — anyone can extend this subtype. |
**Records are implicitly **`**final**`, which is why your snippet compiles without specifying any of those keywords.

#### Explicit `permits` clause
You can write the permitted subtypes explicitly:

```java
public sealed interface AddToCartResult
    permits AddToCartResult.Success,
            AddToCartResult.OutOfStock,
            AddToCartResult.ProductDiscontinued {
    ...
}
```
When all permitted subtypes are declared **inside the same file** (as in your example), the `permits` clause can be omitted — the compiler infers it.

### 2.2 `record` 
A `record` is a compact, immutable data carrier. This one line:

```java
record OutOfStock(String sku, int available) implements AddToCartResult {}
```
…is shorthand that the compiler expands into roughly:

```java
public static final class OutOfStock implements AddToCartResult {
    private final String sku;
    private final int available;

    public OutOfStock(String sku, int available) {
        this.sku = sku;
        this.available = available;
    }

    public String sku()       { return sku; }
    public int available()    { return available; }

    @Override public boolean equals(Object o) { /* field-by-field equality */ }
    @Override public int hashCode()           { /* derived from fields */ }
    @Override public String toString()        { /* "OutOfStock[sku=..., available=...]" */ }
}
```
Key properties of records:

- All components are `private final` .
- A canonical constructor is generated.
- Accessors are named after the components: `sku()` , not `getSku()` .
- `equals` , `hashCode` , and `toString`  are auto-generated.
- The class is implicitly `final`  — it cannot be subclassed.
You can still:

- Add static factory methods.
- Add instance methods.
- Override the canonical constructor (typically as a **compact constructor** for validation):
```java
record OutOfStock(String sku, int available) implements AddToCartResult {
    public OutOfStock {
        if (available < 0) throw new IllegalArgumentException("available must be >= 0");
    }
}
```
---

## 3. How You Use It at the Call Site
The whole reason this pattern is so good is **pattern matching with **`**switch**` (Java 21+):

```java
AddToCartResult result = cartService.addToCart(userId, sku, qty);

String message = switch (result) {
    case AddToCartResult.Success(Cart cart) ->
        "Added! You now have " + cart.itemCount() + " items.";

    case AddToCartResult.OutOfStock(String s, int available) ->
        "Sorry, only " + available + " left of " + s + ".";

    case AddToCartResult.ProductDiscontinued(String s) ->
        "Product " + s + " is no longer sold.";
};
```
Three things to notice:

1. **No **`**default**` ** branch needed.** Because the interface is sealed, the compiler knows the three cases are exhaustive. If you later add a fourth subtype, **every **`**switch**` ** like this becomes a compile error** until you handle it. This is the killer feature.
2. **Record deconstruction patterns** (`Success(Cart cart)` ) pull fields out directly — no `.cart()`  calls needed.
3. The `switch`  is an **expression** that returns a value.
---

## 4. How You'd Write This Without Sealed/Records
There are several traditional approaches. All of them are worse in specific ways.

### 4.1 The "boolean flags + nullable fields" object
This is what most pre-Java-16 codebases look like.

```java
public final class AddToCartResult {
    private final boolean success;
    private final boolean outOfStock;
    private final boolean discontinued;
    private final Cart cart;            // null unless success
    private final String sku;           // null unless out-of-stock or discontinued
    private final Integer available;    // null unless out-of-stock

    private AddToCartResult(boolean success, boolean outOfStock, boolean discontinued,
                            Cart cart, String sku, Integer available) {
        this.success = success;
        this.outOfStock = outOfStock;
        this.discontinued = discontinued;
        this.cart = cart;
        this.sku = sku;
        this.available = available;
    }

    public static AddToCartResult success(Cart cart) {
        return new AddToCartResult(true, false, false, cart, null, null);
    }
    public static AddToCartResult outOfStock(String sku, int available) {
        return new AddToCartResult(false, true, false, null, sku, available);
    }
    public static AddToCartResult discontinued(String sku) {
        return new AddToCartResult(false, false, true, null, sku, null);
    }

    public boolean isSuccess()       { return success; }
    public boolean isOutOfStock()    { return outOfStock; }
    public boolean isDiscontinued()  { return discontinued; }
    public Cart getCart()            { return cart; }
    public String getSku()           { return sku; }
    public Integer getAvailable()    { return available; }
}
```
Usage:

```java
AddToCartResult r = cartService.addToCart(...);
if (r.isSuccess()) {
    System.out.println("Added: " + r.getCart().itemCount());
} else if (r.isOutOfStock()) {
    System.out.println("Only " + r.getAvailable() + " left of " + r.getSku());
} else if (r.isDiscontinued()) {
    System.out.println(r.getSku() + " discontinued");
}
// What if none of the flags are true? What if two are? Compiler can't tell you.
```
**Problems:**

- Every field is nullable; `NullPointerException`  is one typo away.
- Invalid combinations are representable (`success=true && outOfStock=true` ).
- No exhaustiveness — forget a case and you fail silently at runtime.
- ~50 lines of boilerplate for what records do in 3 lines.
### 4.2 Class hierarchy with an abstract base
Closer to the modern version, but still flawed:

```java
public abstract class AddToCartResult {
    public static class Success extends AddToCartResult {
        private final Cart cart;
        public Success(Cart cart) { this.cart = cart; }
        public Cart getCart() { return cart; }
        // plus equals, hashCode, toString...
    }

    public static class OutOfStock extends AddToCartResult {
        private final String sku;
        private final int available;
        public OutOfStock(String sku, int available) { this.sku = sku; this.available = available; }
        public String getSku()    { return sku; }
        public int getAvailable() { return available; }
        // plus equals, hashCode, toString...
    }

    public static class ProductDiscontinued extends AddToCartResult {
        private final String sku;
        public ProductDiscontinued(String sku) { this.sku = sku; }
        public String getSku() { return sku; }
        // plus equals, hashCode, toString...
    }
}
```
Usage with `instanceof`:

```java
AddToCartResult r = cartService.addToCart(...);
if (r instanceof AddToCartResult.Success s) {
    System.out.println("Added: " + s.getCart().itemCount());
} else if (r instanceof AddToCartResult.OutOfStock o) {
    System.out.println("Only " + o.getAvailable() + " left of " + o.getSku());
} else if (r instanceof AddToCartResult.ProductDiscontinued d) {
    System.out.println(d.getSku() + " discontinued");
} else {
    throw new IllegalStateException("Unknown result"); // <-- the giveaway
}
```
**Problems:**

- Not sealed → anyone in the codebase can write `class WeirdResult extends AddToCartResult`  and your `instanceof`  chain silently breaks.
- You must write a `throw`  for an "impossible" case — the compiler can't prove it's impossible.
- Manual `equals` /`hashCode` /`toString` , or you forget them.
### 4.3 The Visitor pattern
The "classical OO" answer to ADTs before sealed types:

```java
public interface AddToCartResult {
    <R> R accept(Visitor<R> v);

    interface Visitor<R> {
        R visitSuccess(Cart cart);
        R visitOutOfStock(String sku, int available);
        R visitDiscontinued(String sku);
    }

    final class Success implements AddToCartResult {
        private final Cart cart;
        public Success(Cart cart) { this.cart = cart; }
        public <R> R accept(Visitor<R> v) { return v.visitSuccess(cart); }
    }
    // ...OutOfStock, ProductDiscontinued similar
}
```
Usage:

```java
String msg = result.accept(new AddToCartResult.Visitor<String>() {
    public String visitSuccess(Cart cart)               { return "Added!"; }
    public String visitOutOfStock(String sku, int avail){ return "Only " + avail + " left"; }
    public String visitDiscontinued(String sku)         { return sku + " gone"; }
});
```
**This actually does give you exhaustiveness** (you must implement all visitor methods). It's why this pattern existed. But:

- It's enormous boilerplate.
- Adding a new case forces edits across every `Visitor`  implementation, which can be the right tradeoff or the wrong one.
- It hides simple control flow behind dynamic dispatch.
The sealed-interface + records + pattern-match `switch` combo gives you the **same exhaustiveness guarantee** for ~10% of the code.

---

## 5. Why the Modern Version Wins — Concretely
| Concern | Legacy flags | Class hierarchy | Visitor | Sealed + Records |
| ----- | ----- | ----- | ----- | ----- |
| Lines of code | ~50 | ~60 | ~80 | **3** |
| Compiler-checked exhaustiveness | ❌ | ❌ | ✅ | ✅ |
| Immutable by default | manual | manual | manual | ✅ |
| `equals`/`hashCode`/`toString`  | manual | manual | manual | ✅ |
| Illegal states unrepresentable | ❌ | ⚠️ | ✅ | ✅ |
| Closed hierarchy enforced | n/a | ❌ | ❌ | ✅ |
| Field extraction at use-site | getters | getters + cast | dispatch method | **deconstruction patterns** |
The real wins are the two that show up in production:

1. **Exhaustiveness as a refactoring tool.** Add a new case (`record PaymentRequired(...) implements AddToCartResult {}` ) and the compiler immediately tells you every `switch`  that needs updating. You cannot forget one.
2. **Illegal states unrepresentable.** There is literally no way to construct a `Success`  that also has a SKU but no cart. The type system makes it impossible, so you don't write defensive `if (cart != null && success)`  everywhere.
---

## 6. When _Not_ to Use This Pattern
Sealed types and records are not free of tradeoffs:

- **Open extension points.** If you're designing a plugin API where third parties should add their own cases, sealing is the wrong call — use a regular interface.
- **Many cases (>10) with little shared logic.** A sealed hierarchy starts feeling clumsy; consider a strategy map or polymorphism instead.
- **Mutable state.** Records are immutable. If you genuinely need mutable fields, records aren't the right tool.
- **Pre-Java 17.** Sealed types require Java 17. Records require Java 16. If you're stuck on Java 8/11, you're back to the legacy options.
---

## 7. Quick Reference Cheat Sheet
```java
// Define
public sealed interface Result<T>
        permits Result.Ok, Result.Err {           // 'permits' optional if same file
    record Ok<T>(T value) implements Result<T> {}
    record Err<T>(String message) implements Result<T> {}
}

// Construct
Result<Cart> r = new Result.Ok<>(cart);

// Consume — exhaustive, no default needed
return switch (r) {
    case Result.Ok<Cart>(Cart c)     -> c.itemCount();
    case Result.Err<Cart>(String m)  -> { log.warn(m); yield 0; }
};
```
---

**TL;DR:** The snippet you posted compresses an entire algebraic-data-type definition — three immutable variants with equality, a closed hierarchy, and compiler-enforced exhaustive handling — into four lines. The pre-Java-16 equivalents needed 50–80 lines and still couldn't give you the exhaustiveness guarantee that lets you refactor without fear. That's the whole sales pitch.



<!--- Eraser file: https://app.eraser.io/workspace/mWDathqgEgnZNqRzdvv4 --->