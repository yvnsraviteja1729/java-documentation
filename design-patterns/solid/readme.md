<p><a target="_blank" href="https://app.eraser.io/workspace/Y1Jdor6FTOKpZuI9PNHY" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>

# SOLID Principles — Java Reference Guide
SOLID is a set of five object-oriented design principles that, when applied together, produce systems that are easier to maintain, extend, and test. The acronym was popularized by Robert C. Martin ("Uncle Bob").

>  All examples in this guide use a consistent **e-commerce order management** domain so you can see how the principles reinforce each other. 

---

## S — Single Responsibility Principle (SRP)
>  **"A class should have only one reason to change."** 

A class should do one thing and do it well. If a class is responsible for multiple concerns, a change in one concern may inadvertently break the other.

### ❌ Violation
```java
// This class has THREE reasons to change:
// 1. Order business logic changes
// 2. Email format changes
// 3. Database schema changes
public class OrderService {

    public void placeOrder(Order order) {
        // Business logic
        order.setStatus("CONFIRMED");
        order.setTotal(calculateTotal(order));

        // Database concern — should not be here
        String sql = "INSERT INTO orders VALUES (?, ?, ?)";
        // ... JDBC logic ...

        // Notification concern — should not be here
        String body = "Dear " + order.getCustomerName() + ", your order is confirmed.";
        // ... email sending logic ...
    }

    private double calculateTotal(Order order) { /* ... */ return 0; }
}
```
### ✅ Correct Application
```java
// Reason to change: only order business logic
public class OrderService {
    private final OrderRepository repository;
    private final NotificationService notificationService;

    public OrderService(OrderRepository repository, NotificationService notificationService) {
        this.repository = repository;
        this.notificationService = notificationService;
    }

    public void placeOrder(Order order) {
        order.setStatus("CONFIRMED");
        order.setTotal(calculateTotal(order));
        repository.save(order);
        notificationService.sendOrderConfirmation(order);
    }

    private double calculateTotal(Order order) {
        return order.getItems().stream()
                .mapToDouble(item -> item.getPrice() * item.getQuantity())
                .sum();
    }
}

// Reason to change: only database/persistence logic
public class OrderRepository {
    public void save(Order order) {
        System.out.println("[DB] Persisting order: " + order.getId());
        // JDBC / JPA logic here
    }

    public Order findById(String id) {
        System.out.println("[DB] Fetching order: " + id);
        return new Order(); // Stub
    }
}

// Reason to change: only email/notification format
public class NotificationService {
    public void sendOrderConfirmation(Order order) {
        String body = "Dear " + order.getCustomerName() + ", your order " +
                order.getId() + " is confirmed. Total: $" + order.getTotal();
        System.out.println("[EMAIL] Sending to " + order.getCustomerEmail() + ": " + body);
    }
}
```
**Why it matters:** When the email template changes, you touch only `NotificationService`. When the DB schema changes, you touch only `OrderRepository`. Neither change can break the other.

---

## O — Open/Closed Principle (OCP)
>  **"Software entities should be open for extension, but closed for modification."** 

You should be able to add new behavior without changing existing, tested code. This is achieved through abstraction (interfaces, abstract classes).

### ❌ Violation
```java
// Every new discount type requires modifying this class — risky!
public class DiscountCalculator {
    public double calculate(Order order, String discountType) {
        if (discountType.equals("SEASONAL")) {
            return order.getTotal() * 0.10;
        } else if (discountType.equals("LOYALTY")) {
            return order.getTotal() * 0.15;
        } else if (discountType.equals("FLASH_SALE")) { // New type = modify this file!
            return order.getTotal() * 0.25;
        }
        return 0;
    }
}
```
### ✅ Correct Application
```java
// Closed for modification — this interface never changes
public interface DiscountStrategy {
    double calculate(Order order);
    String getType();
}

// Open for extension — just add a new class
public class SeasonalDiscount implements DiscountStrategy {
    @Override
    public double calculate(Order order) {
        return order.getTotal() * 0.10;
    }

    @Override
    public String getType() { return "SEASONAL"; }
}

public class LoyaltyDiscount implements DiscountStrategy {
    private final int customerPoints;

    public LoyaltyDiscount(int customerPoints) {
        this.customerPoints = customerPoints;
    }

    @Override
    public double calculate(Order order) {
        double rate = customerPoints > 1000 ? 0.20 : 0.15;
        return order.getTotal() * rate;
    }

    @Override
    public String getType() { return "LOYALTY"; }
}

// Adding a new discount type = zero changes to existing code
public class FlashSaleDiscount implements DiscountStrategy {
    @Override
    public double calculate(Order order) {
        return order.getTotal() * 0.25;
    }

    @Override
    public String getType() { return "FLASH_SALE"; }
}

// This class never needs to change for new discount types
public class CheckoutService {
    public double applyDiscount(Order order, DiscountStrategy strategy) {
        double discount = strategy.calculate(order);
        System.out.printf("Applied %s discount: -$%.2f%n", strategy.getType(), discount);
        return order.getTotal() - discount;
    }
}

// Usage
public class Main {
    public static void main(String[] args) {
        Order order = new Order("ORD-001", 200.0);
        CheckoutService checkout = new CheckoutService();

        System.out.println("Final: $" + checkout.applyDiscount(order, new SeasonalDiscount()));
        System.out.println("Final: $" + checkout.applyDiscount(order, new LoyaltyDiscount(1500)));
        System.out.println("Final: $" + checkout.applyDiscount(order, new FlashSaleDiscount()));
    }
}
```
**Why it matters:** Adding a new promotion (`ChristmasDiscount`, `NewUserDiscount`) is a matter of writing one new class. The `CheckoutService` — already tested and deployed — is never touched.

---

## L — Liskov Substitution Principle (LSP)
>  **"Objects of a subclass should be substitutable for objects of the superclass without breaking the application."** 

If `B` extends `A`, then everywhere you use `A`, you must be able to drop in `B` without the program behaving incorrectly. LSP violations are often subtle and result in unexpected runtime errors.

### ❌ Violation
```java
public class Order {
    public void addItem(OrderItem item) {
        items.add(item);
    }

    public void removeItem(String itemId) {
        items.removeIf(i -> i.getId().equals(itemId));
    }
}

// ArchiveOrder is read-only — it CANNOT support removeItem
// But it extends Order, so callers assume it can!
public class ArchiveOrder extends Order {
    @Override
    public void removeItem(String itemId) {
        // Breaks LSP! Caller using Order reference gets an exception it doesn't expect.
        throw new UnsupportedOperationException("Cannot modify an archived order.");
    }
}

// This silently breaks at runtime:
public void refundItem(Order order, String itemId) {
    order.removeItem(itemId); // Crashes if order is actually an ArchiveOrder!
}
```
### ✅ Correct Application
```java
// Root abstraction captures only what ALL orders share
public interface ReadableOrder {
    String getId();
    List<OrderItem> getItems();
    double getTotal();
    String getStatus();
}

// Mutable orders extend further
public interface MutableOrder extends ReadableOrder {
    void addItem(OrderItem item);
    void removeItem(String itemId);
}

// Active order: fully mutable
public class ActiveOrder implements MutableOrder {
    private final String id;
    private final List<OrderItem> items = new ArrayList<>();
    private String status = "PENDING";

    public ActiveOrder(String id) { this.id = id; }

    @Override public String getId() { return id; }
    @Override public List<OrderItem> getItems() { return Collections.unmodifiableList(items); }
    @Override public double getTotal() {
        return items.stream().mapToDouble(i -> i.getPrice() * i.getQuantity()).sum();
    }
    @Override public String getStatus() { return status; }
    @Override public void addItem(OrderItem item) { items.add(item); }
    @Override public void removeItem(String itemId) {
        items.removeIf(i -> i.getId().equals(itemId));
    }
}

// Archive order: read-only — does NOT claim to be mutable
public class ArchiveOrder implements ReadableOrder {
    private final String id;
    private final List<OrderItem> items;
    private final String status;

    public ArchiveOrder(String id, List<OrderItem> items, String status) {
        this.id = id;
        this.items = List.copyOf(items);
        this.status = status;
    }

    @Override public String getId() { return id; }
    @Override public List<OrderItem> getItems() { return items; }
    @Override public double getTotal() {
        return items.stream().mapToDouble(i -> i.getPrice() * i.getQuantity()).sum();
    }
    @Override public String getStatus() { return status; }
}

// Consumers use the right abstraction
public class OrderReportService {
    // Works with ALL orders safely
    public void printSummary(ReadableOrder order) {
        System.out.printf("Order %s | Status: %s | Total: $%.2f%n",
                order.getId(), order.getStatus(), order.getTotal());
    }
}

public class RefundService {
    // Only accepts orders that can be modified
    public void removeItem(MutableOrder order, String itemId) {
        order.removeItem(itemId);
        System.out.println("Item " + itemId + " removed from order " + order.getId());
    }
}
```
**Why it matters:** `ArchiveOrder` and `ActiveOrder` can both be passed to `OrderReportService` safely. `RefundService` only accepts `MutableOrder` — the compiler prevents you from passing an `ArchiveOrder` there.

---

## I — Interface Segregation Principle (ISP)
>  **"Clients should not be forced to depend on interfaces they do not use."** 

Split large, fat interfaces into smaller, focused ones. This prevents a class from being forced to implement methods that are irrelevant to its role.

### ❌ Violation
```java
// One giant interface that no single class fully needs
public interface OrderProcessor {
    void validateOrder(Order order);
    void processPayment(Order order);
    void generateInvoice(Order order);
    void shipOrder(Order order);
    void sendTrackingEmail(Order order);
    void handleReturn(Order order);    // Warehouse doesn't do this
    void issueRefund(Order order);     // Warehouse doesn't do this
}

// Warehouse is forced to provide empty stubs for finance methods
public class WarehouseService implements OrderProcessor {
    @Override public void validateOrder(Order order) { /* real logic */ }
    @Override public void shipOrder(Order order) { /* real logic */ }
    @Override public void processPayment(Order order) { /* NOT MY JOB — stub! */ }
    @Override public void generateInvoice(Order order) { /* NOT MY JOB — stub! */ }
    @Override public void sendTrackingEmail(Order order) { /* NOT MY JOB — stub! */ }
    @Override public void handleReturn(Order order) { /* NOT MY JOB — stub! */ }
    @Override public void issueRefund(Order order) { /* NOT MY JOB — stub! */ }
}
```
### ✅ Correct Application
```java
// Small, focused interfaces — each describes one role
public interface OrderValidator {
    void validateOrder(Order order);
}

public interface PaymentProcessor {
    void processPayment(Order order);
    void issueRefund(Order order);
}

public interface InvoiceGenerator {
    void generateInvoice(Order order);
}

public interface ShippingHandler {
    void shipOrder(Order order);
    void sendTrackingEmail(Order order);
}

public interface ReturnHandler {
    void handleReturn(Order order);
}

// Each service implements ONLY what it needs
public class WarehouseService implements OrderValidator, ShippingHandler {
    @Override
    public void validateOrder(Order order) {
        System.out.println("[Warehouse] Validating stock for order: " + order.getId());
    }

    @Override
    public void shipOrder(Order order) {
        System.out.println("[Warehouse] Shipping order: " + order.getId());
    }

    @Override
    public void sendTrackingEmail(Order order) {
        System.out.println("[Warehouse] Sending tracking info for: " + order.getId());
    }
}

public class FinanceService implements PaymentProcessor, InvoiceGenerator {
    @Override
    public void processPayment(Order order) {
        System.out.println("[Finance] Processing payment for: " + order.getId());
    }

    @Override
    public void issueRefund(Order order) {
        System.out.println("[Finance] Refunding order: " + order.getId());
    }

    @Override
    public void generateInvoice(Order order) {
        System.out.println("[Finance] Invoice generated for: " + order.getId());
    }
}

public class ReturnService implements ReturnHandler {
    @Override
    public void handleReturn(Order order) {
        System.out.println("[Returns] Processing return for: " + order.getId());
    }
}
```
**Why it matters:** `WarehouseService` has no knowledge of payment or refund logic. If `PaymentProcessor` changes, `WarehouseService` is completely unaffected and doesn't need to recompile.

---

## D — Dependency Inversion Principle (DIP)
>  **"High-level modules should not depend on low-level modules. Both should depend on abstractions."** 

The core business logic should not be hard-wired to concrete implementations (MySQL, SendGrid, etc.). It should depend on interfaces, so implementations can be swapped, tested, or extended freely.

### ❌ Violation
```java
// High-level service is tightly coupled to concrete low-level classes
public class OrderService {

    // Hard dependency on a specific DB implementation
    private MySqlOrderRepository repository = new MySqlOrderRepository();

    // Hard dependency on a specific email provider
    private SendGridEmailService emailService = new SendGridEmailService();

    public void placeOrder(Order order) {
        repository.save(order);          // Breaks if you switch to PostgreSQL
        emailService.sendEmail(order);   // Breaks if you switch to Mailchimp
    }
}
```
### ✅ Correct Application
```java
// Abstractions that both high and low level modules depend on
public interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(String id);
}

public interface EmailService {
    void sendOrderConfirmation(Order order);
}

// Low-level modules implement the abstractions
public class MySqlOrderRepository implements OrderRepository {
    @Override
    public void save(Order order) {
        System.out.println("[MySQL] Saved order: " + order.getId());
    }

    @Override
    public Optional<Order> findById(String id) {
        System.out.println("[MySQL] Querying order: " + id);
        return Optional.empty(); // Stub
    }
}

public class PostgresOrderRepository implements OrderRepository {
    @Override
    public void save(Order order) {
        System.out.println("[Postgres] Saved order: " + order.getId());
    }

    @Override
    public Optional<Order> findById(String id) {
        System.out.println("[Postgres] Querying order: " + id);
        return Optional.empty(); // Stub
    }
}

public class SendGridEmailService implements EmailService {
    @Override
    public void sendOrderConfirmation(Order order) {
        System.out.println("[SendGrid] Email sent for order: " + order.getId());
    }
}

public class MailchimpEmailService implements EmailService {
    @Override
    public void sendOrderConfirmation(Order order) {
        System.out.println("[Mailchimp] Email sent for order: " + order.getId());
    }
}

// High-level module depends ONLY on abstractions
public class OrderService {
    private final OrderRepository repository;
    private final EmailService emailService;

    // Dependencies are injected (Spring @Autowired, or constructor injection)
    public OrderService(OrderRepository repository, EmailService emailService) {
        this.repository = repository;
        this.emailService = emailService;
    }

    public void placeOrder(Order order) {
        order.setStatus("CONFIRMED");
        repository.save(order);
        emailService.sendOrderConfirmation(order);
        System.out.println("Order " + order.getId() + " placed successfully.");
    }
}

// Usage / Composition Root (or Spring @Configuration)
public class Main {
    public static void main(String[] args) {
        // Swap implementations here without touching OrderService
        OrderRepository repo = new PostgresOrderRepository();
        EmailService email   = new SendGridEmailService();
        OrderService service = new OrderService(repo, email);

        Order order = new Order("ORD-001", "Jane Doe");
        service.placeOrder(order);
    }
}

// For unit tests — use an in-memory fake, no real DB or email needed
class InMemoryOrderRepository implements OrderRepository {
    private final Map<String, Order> store = new HashMap<>();

    @Override
    public void save(Order order) { store.put(order.getId(), order); }

    @Override
    public Optional<Order> findById(String id) { return Optional.ofNullable(store.get(id)); }
}
```
**Why it matters:** `OrderService` doesn't care whether the database is MySQL, Postgres, or in-memory. You can test it with `InMemoryOrderRepository` and deploy it with `MySqlOrderRepository` — without changing a single line of business logic.

---

## SOLID Principles — Quick Reference
| Principle | One-liner | Key Mechanism | Violation Smell |
| ----- | ----- | ----- | ----- |
| **SRP** | One class, one job | Separate classes per concern | "God class" with unrelated methods |
| **OCP** | Extend, don't modify | Interfaces + new implementations | `if-else` chains growing with new types |
| **LSP** | Subclasses are drop-in replacements | Correct inheritance hierarchies | `UnsupportedOperationException` in subclass |
| **ISP** | Small, focused interfaces | Split fat interfaces by role | Classes with empty/stub method bodies |
| **DIP** | Depend on abstractions | Constructor injection of interfaces | `new ConcreteClass()` inside business logic |




<!--- Eraser file: https://app.eraser.io/workspace/Y1Jdor6FTOKpZuI9PNHY --->