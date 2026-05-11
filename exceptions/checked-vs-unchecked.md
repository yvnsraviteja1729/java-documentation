<p><a target="_blank" href="https://app.eraser.io/workspace/DgLvRwkiQqaLARXOfs99" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>

A practical, opinionated guide to exception handling using a real-world e-commerce domain (orders, payments, inventory, shipping).

---

## 1. The Exception Hierarchy (Refresher)
```
Throwable
├── Error                    ← JVM-level, never catch (OutOfMemoryError, StackOverflowError)
└── Exception                ← CHECKED (compiler enforces handling)
    └── RuntimeException     ← UNCHECKED (compiler ignores)
```
| Category | Examples | Use For |
| ----- | ----- | ----- |
| `Error`  | `OutOfMemoryError`  | Don't catch. Let JVM die. |
| Checked `Exception`  | `IOException`, `SQLException`  | Recoverable, external failures |
| `RuntimeException`  | `NullPointerException`, `IllegalArgumentException`  | Programming bugs, preconditions |
---

## 2. The Decision Framework
Before throwing any exception, answer these in order:

```
1. Is this a programming bug (null, invalid arg, illegal state)?
       → RuntimeException (unchecked)
2. Is this a normal business outcome (out of stock, declined card)?
       → Prefer a Result type, OR a domain-specific unchecked exception
3. Is this an external/environmental failure the caller MUST handle
   (DB down, payment gateway timeout, file missing)?
       → Checked exception OR wrap as unchecked at the boundary
4. Is this an exceptional, unrecoverable system failure?
       → Unchecked, propagate to a global handler
```
---

## 3. E-commerce Domain Exception Design
### 3.1 Build a Domain Exception Hierarchy
Create a **base exception** per bounded context. This makes catching, logging, and HTTP mapping clean.

```java
/**
 * Base class for all expected business errors in the e-commerce domain.
 * Unchecked by design — handled centrally at the API boundary.
 */
public abstract class BusinessException extends RuntimeException {
    private final String errorCode;

    protected BusinessException(String errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
    }

    protected BusinessException(String errorCode, String message, Throwable cause) {
        super(message, cause);
        this.errorCode = errorCode;
    }

    public String getErrorCode() { return errorCode; }
}
```
```java
// ---- Catalog ----
public class ProductNotFoundException extends BusinessException {
    public ProductNotFoundException(String sku) {
        super("PRODUCT_NOT_FOUND", "Product not found: " + sku);
    }
}

// ---- Inventory ----
public class OutOfStockException extends BusinessException {
    private final String sku;
    private final int requested;
    private final int available;

    public OutOfStockException(String sku, int requested, int available) {
        super("OUT_OF_STOCK",
              String.format("SKU %s: requested %d, available %d", sku, requested, available));
        this.sku = sku;
        this.requested = requested;
        this.available = available;
    }
    // getters...
}

// ---- Cart ----
public class CartEmptyException extends BusinessException {
    public CartEmptyException(String cartId) {
        super("CART_EMPTY", "Cannot checkout empty cart: " + cartId);
    }
}

// ---- Pricing / Promo ----
public class InvalidCouponException extends BusinessException {
    public InvalidCouponException(String code, String reason) {
        super("INVALID_COUPON", "Coupon " + code + " invalid: " + reason);
    }
}

// ---- Payment ----
public class PaymentDeclinedException extends BusinessException {
    private final String declineCode; // from gateway: "insufficient_funds", "do_not_honor"
    public PaymentDeclinedException(String declineCode, String message) {
        super("PAYMENT_DECLINED", message);
        this.declineCode = declineCode;
    }
    public String getDeclineCode() { return declineCode; }
}

public class PaymentGatewayException extends BusinessException {
    // For technical failures (timeout, 5xx from Stripe, etc.)
    public PaymentGatewayException(String message, Throwable cause) {
        super("PAYMENT_GATEWAY_ERROR", message, cause);
    }
}

// ---- Order ----
public class OrderNotFoundException extends BusinessException {
    public OrderNotFoundException(String orderId) {
        super("ORDER_NOT_FOUND", "Order not found: " + orderId);
    }
}

public class IllegalOrderStateException extends BusinessException {
    // e.g., trying to cancel an already-shipped order
    public IllegalOrderStateException(String orderId, String currentState, String attempted) {
        super("ILLEGAL_ORDER_STATE",
              String.format("Order %s in state %s cannot %s", orderId, currentState, attempted));
    }
}

// ---- Auth ----
public class UnauthorizedException extends BusinessException {
    public UnauthorizedException(String message) { super("UNAUTHORIZED", message); }
}
```
**Why unchecked?** In a Spring/modern stack, you handle these centrally (Section 5). Forcing every service layer to declare `throws OutOfStockException, PaymentDeclinedException, ...` adds noise without benefit.

---

## 4. Real-World Scenarios — When to Use What
### Scenario 1: Product lookup by SKU
```java
public Product findBySku(String sku) {
    if (sku == null || sku.isBlank()) {
        // Programming bug — caller violated contract
        throw new IllegalArgumentException("SKU must not be blank");
    }
    return productRepository.findBySku(sku)
        .orElseThrow(() -> new ProductNotFoundException(sku));  // domain exception
}
```
- `IllegalArgumentException`  → **bug in caller code** (precondition).
- `ProductNotFoundException`  → **expected business outcome** (user typed wrong URL).
### Scenario 2: Add to cart — use a Result type, not an exception
"Item out of stock" when adding to cart is **routine, not exceptional**. Returning a result is clearer:

```java
public sealed interface AddToCartResult {
    record Success(Cart cart) implements AddToCartResult {}
    record OutOfStock(String sku, int available) implements AddToCartResult {}
    record ProductDiscontinued(String sku) implements AddToCartResult {}
}
```
```java
public AddToCartResult addItem(String cartId, String sku, int qty) {
    Product p = productService.findBySku(sku);
    if (p.isDiscontinued()) return new AddToCartResult.ProductDiscontinued(sku);

    int available = inventoryService.available(sku);
    if (available < qty) return new AddToCartResult.OutOfStock(sku, available);

    Cart cart = cartRepository.find(cartId);
    cart.add(p, qty);
    return new AddToCartResult.Success(cartRepository.save(cart));
}
```
**Rule of thumb:** if the failure is _informational and recoverable inline_, a Result type beats an exception.

### Scenario 3: Checkout — multiple failure modes via exceptions
Checkout has many failure modes that must abort the flow. Exceptions are appropriate here because each one short-circuits the orchestration:

```java
@Transactional
public Order checkout(String cartId, PaymentDetails payment) {
    Cart cart = cartRepository.find(cartId);
    if (cart.isEmpty()) {
        throw new CartEmptyException(cartId);
    }

    // Reserve inventory — throws OutOfStockException if any item unavailable
    inventoryService.reserve(cart.items());

    // Charge payment — throws PaymentDeclinedException or PaymentGatewayException
    PaymentReceipt receipt = paymentService.charge(cart.total(), payment);

    Order order = Order.create(cart, receipt);
    return orderRepository.save(order);
    // If anything throws, @Transactional rolls back the DB,
    // but you ALSO need compensating actions for external calls (Section 6).
}
```
### Scenario 4: Payment gateway — wrap external failures
This is the **classic place to convert checked → unchecked** at an integration boundary.

```java
public PaymentReceipt charge(Money amount, PaymentDetails details) {
    try {
        Charge charge = Charge.create(buildParams(amount, details));  // Stripe SDK
        if (!"succeeded".equals(charge.getStatus())) {
            throw new PaymentDeclinedException(
                charge.getFailureCode(),
                charge.getFailureMessage());
        }
        return new PaymentReceipt(charge.getId(), amount);

    } catch (CardException e) {
        // Stripe checked exception → translate to domain language
        throw new PaymentDeclinedException(e.getCode(), e.getMessage());

    } catch (StripeException e) {
        // Network, 5xx, rate limit — caller can retry
        throw new PaymentGatewayException("Stripe call failed", e);
    }
}
```
Key principles applied:

- **Don't leak third-party exceptions** into your domain.
- **Preserve the cause** (`new ...(message, e)` ) so stack traces survive.
- **Distinguish declined (business) vs. gateway error (technical)** — different retry semantics.
### Scenario 5: Order cancellation — state machine violations
```java
public void cancel() {
    if (status == OrderStatus.SHIPPED || status == OrderStatus.DELIVERED) {
        throw new IllegalOrderStateException(id, status.name(), "cancel");
    }
    this.status = OrderStatus.CANCELLED;
}
```
`IllegalStateException` (or domain subclass) is perfect for **invariant violations on an object's state**.

### Scenario 6: Input validation at the API edge
```java
public record PlaceOrderRequest(
    @NotBlank String cartId,
    @NotNull @Valid PaymentDetails payment,
    @Email String customerEmail
) {}
```
Use **Bean Validation** (`jakarta.validation`) instead of throwing `IllegalArgumentException` manually. It produces structured errors automatically.

### Scenario 7: Concurrent inventory update — retry-worthy exception
```java
@Retryable(retryFor = OptimisticLockException.class, maxAttempts = 3)
public void reserve(List<OrderItem> items) {
    for (OrderItem item : items) {
        InventoryRow row = inventoryRepo.findBySkuForUpdate(item.sku());
        if (row.available() < item.qty()) {
            throw new OutOfStockException(item.sku(), item.qty(), row.available());
        }
        row.reserve(item.qty());
        inventoryRepo.save(row);  // may throw OptimisticLockException → retry
    }
}
```
Distinguish:

- **Retryable** (transient): `OptimisticLockException` , `PaymentGatewayException` , timeouts.
- **Non-retryable** (business): `OutOfStockException` , `PaymentDeclinedException` .
---

## 5. Centralized Exception Handling (Spring Boot)
Don't sprinkle `try-catch` in controllers. Use `@RestControllerAdvice`:

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    // ----- 404s -----
    @ExceptionHandler({ProductNotFoundException.class, OrderNotFoundException.class})
    public ResponseEntity<ErrorResponse> handleNotFound(BusinessException ex) {
        return build(HttpStatus.NOT_FOUND, ex);
    }

    // ----- 409 Conflict -----
    @ExceptionHandler({OutOfStockException.class, IllegalOrderStateException.class})
    public ResponseEntity<ErrorResponse> handleConflict(BusinessException ex) {
        return build(HttpStatus.CONFLICT, ex);
    }

    // ----- 402 Payment Required -----
    @ExceptionHandler(PaymentDeclinedException.class)
    public ResponseEntity<ErrorResponse> handlePaymentDeclined(PaymentDeclinedException ex) {
        return build(HttpStatus.PAYMENT_REQUIRED, ex);
    }

    // ----- 502 Bad Gateway (external system failure) -----
    @ExceptionHandler(PaymentGatewayException.class)
    public ResponseEntity<ErrorResponse> handleGateway(PaymentGatewayException ex) {
        log.error("Payment gateway failure", ex);  // log full stack — it's a real incident
        return build(HttpStatus.BAD_GATEWAY, ex);
    }

    // ----- 400 Validation -----
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        List<String> errors = ex.getBindingResult().getFieldErrors().stream()
            .map(f -> f.getField() + ": " + f.getDefaultMessage())
            .toList();
        return ResponseEntity.badRequest()
            .body(new ErrorResponse("VALIDATION_FAILED", "Invalid request", errors));
    }

    // ----- 401 / 403 -----
    @ExceptionHandler(UnauthorizedException.class)
    public ResponseEntity<ErrorResponse> handleAuth(UnauthorizedException ex) {
        return build(HttpStatus.UNAUTHORIZED, ex);
    }

    // ----- Catch-all 500 -----
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleUnexpected(Exception ex) {
        log.error("Unhandled exception", ex);
        return build(HttpStatus.INTERNAL_SERVER_ERROR,
            new ErrorResponse("INTERNAL_ERROR", "Something went wrong", List.of()));
    }

    private ResponseEntity<ErrorResponse> build(HttpStatus status, BusinessException ex) {
        log.warn("{}: {}", ex.getErrorCode(), ex.getMessage());
        return ResponseEntity.status(status)
            .body(new ErrorResponse(ex.getErrorCode(), ex.getMessage(), List.of()));
    }
}

public record ErrorResponse(String code, String message, List<String> details) {}
```
Now every controller is clean:

```java
@PostMapping("/checkout")
public OrderDto checkout(@Valid @RequestBody PlaceOrderRequest req) {
    return OrderDto.from(checkoutService.checkout(req.cartId(), req.payment()));
    // No try-catch. Exceptions bubble to GlobalExceptionHandler.
}
```
---

## 6. Advanced Patterns
### 6.1 Don't use exceptions for control flow
❌ **Bad:**

```java
try {
    return userRepository.findById(id);
} catch (UserNotFoundException e) {
    return createGuestUser();  // exception drives normal flow
}
```
✅ **Good:**

```java
return userRepository.findById(id).orElseGet(this::createGuestUser);
```
### 6.2 Always preserve the cause when wrapping
❌ `throw new PaymentGatewayException("Stripe failed");` — stack trace lost
✅ `throw new PaymentGatewayException("Stripe failed", e);` — chain preserved

### 6.3 Add context, not just rethrow
```java
catch (SQLException e) {
    throw new OrderPersistenceException(
        "Failed to save order " + order.getId() + " for customer " + order.getCustomerId(),
        e);
}
```
### 6.4 Compensating actions on failure (Saga pattern)
In distributed e-commerce, a `@Transactional` annotation can't roll back an external payment charge. Use try/catch to compensate:

```java
public Order checkout(String cartId, PaymentDetails pd) {
    ReservationId reservation = inventoryService.reserve(cart.items());
    try {
        PaymentReceipt receipt = paymentService.charge(cart.total(), pd);
        try {
            return orderRepository.save(Order.create(cart, receipt, reservation));
        } catch (RuntimeException e) {
            paymentService.refund(receipt);     // compensate
            inventoryService.release(reservation);
            throw e;
        }
    } catch (RuntimeException e) {
        inventoryService.release(reservation);  // compensate
        throw e;
    }
}
```
### 6.5 try-with-resources for cleanup
```java
try (Connection conn = dataSource.getConnection();
     PreparedStatement ps = conn.prepareStatement(sql)) {
    // ...
} catch (SQLException e) {
    throw new OrderPersistenceException("Failed to load orders", e);
}
```
### 6.6 Multi-catch for parallel handling
```java
try {
    shippingClient.createLabel(order);
} catch (TimeoutException | IOException e) {
    throw new ShippingProviderException("FedEx unavailable", e);
}
```
### 6.7 Don't catch what you can't handle
```java
// ❌ Useless — just rethrowing
try { ... } catch (Exception e) { throw e; }

// ❌ Disastrous — silent swallow
try { ... } catch (Exception e) { /* nothing */ }

// ✅ Either handle meaningfully or let it propagate
```
---

## 7. Checked vs Unchecked — E-commerce Decisions
| Scenario | Type | Why |
| ----- | ----- | ----- |
| SKU is null | `IllegalArgumentException` (unchecked) | Bug — caller violated contract |
| Product not in DB | `ProductNotFoundException` (unchecked domain) | Maps cleanly to HTTP 404 via advice |
| Add to cart, out of stock | `AddToCartResult.OutOfStock` (no exception) | Inline recovery is normal |
| Checkout, out of stock | `OutOfStockException` (unchecked domain) | Aborts orchestration |
| Card declined | `PaymentDeclinedException` (unchecked domain) | Business outcome → HTTP 402 |
| Stripe API timeout | `PaymentGatewayException` (unchecked domain) | Technical → HTTP 502 + retry |
| Cancel shipped order | `IllegalOrderStateException` (unchecked domain) | Invariant violation → HTTP 409 |
| DB connection lost | Let `DataAccessException` propagate | Spring already wraps; advice handles 500 |
| File upload of product image fails | `ImageUploadException` (could be checked) | If caller has a clear retry path |
---

## 8. Logging Strategy
| Level | When | Example |
| ----- | ----- | ----- |
| `WARN`  | Expected business exception | `PaymentDeclinedException`, `OutOfStockException` — log message only |
| `ERROR`  | Unexpected / external failure | `PaymentGatewayException`, uncaught `Exception` — log full stack |
| `INFO`  | Successful business event | Order placed |
| `DEBUG`  | Diagnostic detail | Request/response payloads |
Rule: **log an exception exactly once**, at the boundary where it's handled. Don't log-and-rethrow at every layer (creates duplicate noise).

---

## 9. Anti-Patterns Checklist
- ❌ Empty `catch`  blocks
- ❌ Catching `Throwable`  or `Exception`  broadly (except in global handler)
- ❌ Using exceptions for normal control flow
- ❌ Throwing raw `Exception`  or `RuntimeException`  — always be specific
- ❌ Losing the cause (`throw new X("...")`  instead of `throw new X("...", e)` )
- ❌ Logging and rethrowing at every layer
- ❌ Declaring `throws Exception`  on every method
- ❌ Returning `null`  instead of throwing or using `Optional` 
- ❌ Catching `NullPointerException`  to "handle" nulls — fix the null source
---

## 10. TL;DR Cheat Sheet
```
┌─ Programming bug?         → IllegalArgumentException / IllegalStateException / NPE
├─ Routine business outcome? → Result type (sealed interface / Optional)
├─ Domain rule violation?    → Custom unchecked BusinessException subclass
├─ External system failure?  → Wrap as unchecked at boundary, preserve cause
└─ Handle WHERE?             → Centrally, in @RestControllerAdvice (Spring)
                              or one outer try-catch (plain Java)
```
**Golden rules:**

1. Exceptions describe _what_ went wrong, not _how to recover_. Recovery is the caller's job.
2. Be specific — one exception type per failure mode.
3. Translate exceptions at architectural boundaries.
4. Always preserve the cause when wrapping.
5. Handle centrally, not everywhere.






<!--- Eraser file: https://app.eraser.io/workspace/DgLvRwkiQqaLARXOfs99 --->