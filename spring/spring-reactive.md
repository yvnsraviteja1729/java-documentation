# Reactive Programming in Java with Spring WebFlux — Detailed End-to-End Notes

---

## 🧭 1. Introduction to Reactive Programming

**Reactive Programming** is a **programming paradigm** focused on **asynchronous, non-blocking, event-driven** data streams.

Instead of *pulling* data (like traditional code), you **subscribe** to streams and **react** when data arrives.

### 🔹 Core Principles (Reactive Manifesto)

| Principle | Meaning |
|-----------|---------|
| **Responsive** | Quick response under all conditions |
| **Resilient** | Stays responsive during failure |
| **Elastic** | Scales up/down based on load |
| **Message-driven** | Async, non-blocking communication |

---

## 🧱 2. Traditional vs Reactive Model

| Aspect | Traditional (Servlet / Spring MVC) | Reactive (WebFlux) |
|--------|-----------------------------------|--------------------|
| Threading | **1 request = 1 thread** (blocking) | **Few threads** handle many requests (non-blocking) |
| I/O | Blocking | Non-blocking |
| Backpressure | ❌ | ✅ |
| Throughput | Limited | Very high |
| Best for | CPU-heavy, small concurrency | I/O-heavy, high concurrency |

### Example
- **Traditional:** 200 concurrent users → 200 threads.
- **Reactive:** 10,000 concurrent users → handled by ~10 event-loop threads.

---

## ⚙️ 3. What is Spring WebFlux?

**Spring WebFlux** is the **reactive web framework** introduced in **Spring 5**, built on top of **Project Reactor**, running on **Netty** (default), Undertow, or Servlet 3.1+ containers.

### Built On
- **Project Reactor** → core library (`Mono`, `Flux`)
- **Reactive Streams API** → standard (`Publisher`, `Subscriber`, `Subscription`, `Processor`)
- **Netty** → non-blocking server

---

## 🧩 4. Reactive Streams Specification

Defines 4 interfaces in `org.reactivestreams`:

```java
Publisher<T>      // emits data
Subscriber<T>     // consumes data
Subscription      // controls flow (request/cancel)
Processor<T,R>    // both Publisher + Subscriber
```

### Flow
```
Publisher  →  Subscription  →  Subscriber
        emits      controls         reacts
```

---

## 🔄 5. Core Reactor Types — `Mono` & `Flux`

### 🔹 `Mono<T>` → 0 or 1 element
```java
Mono<String> mono = Mono.just("Hello");
mono.subscribe(System.out::println);   // Hello
```

### 🔹 `Flux<T>` → 0 to N elements
```java
Flux<String> flux = Flux.just("A", "B", "C");
flux.subscribe(System.out::println);   // A B C
```

### Real-World Analogy
| Type | Example |
|------|---------|
| `Mono` | Fetch a single user by ID |
| `Flux` | Stream all users / live stock prices |

---

## 🧠 6. How Reactive Works Internally

```
Client → Request → Netty Event Loop → Handler returns Mono/Flux
                                          │
                                          ▼
                                   Subscriber subscribes
                                          │
                                          ▼
                                 Data emitted asynchronously
                                          │
                                          ▼
                                 Backpressure handled
```

> Few **event-loop threads** handle thousands of connections — no thread is ever blocked waiting for I/O.

---

## 💡 7. Creating Mono and Flux

```java
Mono.just("Java");
Mono.empty();
Mono.error(new RuntimeException("Oops"));

Flux.just(1, 2, 3);
Flux.range(1, 5);                       // 1..5
Flux.fromIterable(List.of("a","b"));
Flux.interval(Duration.ofSeconds(1));   // emits every 1s
```

---

## 🔧 8. Common Operators

| Operator | Purpose | Example |
|----------|---------|---------|
| `map` | Transform each element | `flux.map(n -> n*2)` |
| `flatMap` | Async transform → Publisher | `flux.flatMap(this::callApi)` |
| `filter` | Filter elements | `flux.filter(n -> n>10)` |
| `zip` | Combine streams | `Flux.zip(f1, f2)` |
| `merge` | Combine async | `Flux.merge(f1, f2)` |
| `concat` | Sequential combine | `Flux.concat(f1, f2)` |
| `delayElements` | Delay each emit | `flux.delayElements(Duration.ofSeconds(1))` |
| `onErrorReturn` | Fallback value | `mono.onErrorReturn("default")` |
| `retry` | Retry on error | `mono.retry(3)` |
| `subscribeOn` | Run on a Scheduler | `flux.subscribeOn(Schedulers.boundedElastic())` |

### Example
```java
Flux.range(1, 5)
    .map(n -> n * 10)
    .filter(n -> n > 20)
    .subscribe(System.out::println);   // 30 40 50
```

---

## 🌊 9. Backpressure

**Backpressure** = ability of subscriber to tell publisher *how much data it can handle*.

```java
Flux.range(1, 100)
    .onBackpressureBuffer(10)
    .subscribe(new BaseSubscriber<>() {
        @Override
        protected void hookOnSubscribe(Subscription s) { request(5); }

        @Override
        protected void hookOnNext(Integer val) {
            System.out.println(val);
            request(1);
        }
    });
```

### Strategies
- `onBackpressureBuffer()` — buffer overflow data
- `onBackpressureDrop()` — drop new items
- `onBackpressureLatest()` — keep latest only

---

## 🛠️ 10. Setting Up Spring WebFlux

### Maven Dependency
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
```

### Reactive MongoDB Example
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb-reactive</artifactId>
</dependency>
```

---

## 🧩 11. Two Programming Models

### 🔹 (A) Annotation-Based (like Spring MVC)
```java
@RestController
@RequestMapping("/products")
public class ProductController {

    @Autowired
    private ProductService service;

    @GetMapping
    public Flux<Product> getAll() {
        return service.findAll();
    }

    @GetMapping("/{id}")
    public Mono<Product> getById(@PathVariable String id) {
        return service.findById(id);
    }

    @PostMapping
    public Mono<Product> create(@RequestBody Product p) {
        return service.save(p);
    }
}
```

### 🔹 (B) Functional / Router Functions
```java
@Configuration
public class Routes {
    @Bean
    public RouterFunction<ServerResponse> route(ProductHandler handler) {
        return RouterFunctions
            .route(GET("/products"), handler::all)
            .andRoute(GET("/products/{id}"), handler::byId)
            .andRoute(POST("/products"), handler::create);
    }
}
```

---

## 🌍 12. End-to-End Real World Example: Product Service

### 🔸 Entity
```java
@Document
public class Product {
    @Id private String id;
    private String name;
    private double price;
}
```

### 🔸 Repository (Reactive)
```java
public interface ProductRepository
        extends ReactiveMongoRepository<Product, String> {

    Flux<Product> findByPriceGreaterThan(double price);
}
```

### 🔸 Service
```java
@Service
public class ProductService {
    @Autowired private ProductRepository repo;

    public Flux<Product> findAll() { return repo.findAll(); }
    public Mono<Product> findById(String id) { return repo.findById(id); }
    public Mono<Product> save(Product p) { return repo.save(p); }
    public Mono<Void> delete(String id) { return repo.deleteById(id); }

    public Flux<Product> getExpensiveProducts() {
        return repo.findByPriceGreaterThan(1000);
    }
}
```

### 🔸 Controller
```java
@RestController
@RequestMapping("/products")
public class ProductController {
    @Autowired private ProductService service;

    @GetMapping
    public Flux<Product> all() { return service.findAll(); }

    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<Product> stream() {
        return service.findAll().delayElements(Duration.ofSeconds(1));
    }
}
```

➡ The `/stream` endpoint sends data continuously via **Server-Sent Events (SSE)**.

---

## 🔁 13. Reactive `WebClient` (Reactive HTTP Client)

Replaces blocking `RestTemplate`.

```java
WebClient client = WebClient.create("http://api.example.com");

Mono<User> user = client.get()
    .uri("/users/{id}", 1)
    .retrieve()
    .bodyToMono(User.class);

Flux<User> users = client.get()
    .uri("/users")
    .retrieve()
    .bodyToFlux(User.class);
```

### Real-World Use
- Call third-party APIs (payment gateway, weather API).
- Microservice-to-microservice non-blocking calls.

---

## 🔐 14. Error Handling

```java
Mono<User> userMono = repo.findById(id)
    .switchIfEmpty(Mono.error(new UserNotFoundException()))
    .onErrorResume(e -> Mono.just(new User("default")))
    .doOnError(err -> log.error("Error fetching user", err));
```

| Operator | Use |
|----------|-----|
| `onErrorReturn` | Return fallback value |
| `onErrorResume` | Return fallback publisher |
| `onErrorMap` | Translate exception |
| `retry(n)` | Retry n times |
| `doOnError` | Side-effect on error |

---

## 🧵 15. Schedulers (Threading Control)

```java
Flux.range(1, 5)
    .map(i -> heavyTask(i))
    .subscribeOn(Schedulers.boundedElastic())
    .subscribe();
```

| Scheduler | Use |
|-----------|-----|
| `parallel()` | CPU-bound work |
| `boundedElastic()` | Blocking I/O (legacy DB/file) |
| `single()` | Single-thread tasks |
| `immediate()` | Current thread |

---

## 🔄 16. Combining Streams (Real-Time Example)

**Scenario:** Get user info + their orders from two services in parallel.

```java
Mono<User> userMono = webClient.get().uri("/users/1").retrieve().bodyToMono(User.class);
Mono<List<Order>> ordersMono = webClient.get().uri("/orders/user/1")
    .retrieve().bodyToFlux(Order.class).collectList();

Mono<UserProfile> profile = Mono.zip(userMono, ordersMono)
    .map(tuple -> new UserProfile(tuple.getT1(), tuple.getT2()));
```

➡ Both calls happen **concurrently**, then combined when both complete.

---

## ⚡ 17. Real-World Use Cases of WebFlux

| Domain | Use Case |
|--------|----------|
| 💬 Chat Apps | Real-time messaging via SSE / WebSocket |
| 📊 Stock Market | Live price streaming |
| 🛒 E-commerce | High concurrency product browsing |
| 🚖 Ride-Sharing | Real-time driver location updates |
| 🌐 API Gateways | Spring Cloud Gateway (built on WebFlux) |
| 🧠 IoT | Stream sensor data |
| 🏦 Banking | Async transaction processing |

---

## 🧠 18. WebFlux vs Spring MVC

| Feature | Spring MVC | Spring WebFlux |
|---------|------------|----------------|
| Programming model | Imperative | Reactive |
| Threading | Blocking | Non-blocking |
| Server | Tomcat | Netty (default) |
| Return type | `User`, `List<User>` | `Mono<User>`, `Flux<User>` |
| Client | `RestTemplate` | `WebClient` |
| Performance | Limited concurrency | High concurrency |
| Best for | CRUD, simple apps | Streaming, microservices |

---

## ✅ 19. Advantages

- 🚀 High **scalability** with minimal threads
- 🧠 Efficient **resource usage**
- ⚡ Built-in **backpressure**
- 🔁 Excellent for **microservices & streaming**
- 🧩 Composable **functional pipelines**

## ❌ Disadvantages

- 🧠 Steeper **learning curve**
- 🧪 Harder **debugging** (stack traces)
- 🚫 Not ideal for **CPU-heavy** workloads
- 🧵 Must avoid **blocking calls** inside reactive flow

---

## 🧭 20. Best Practices

✅ Never call `.block()` inside reactive chains.
✅ Use **`WebClient`** instead of `RestTemplate`.
✅ Use **reactive drivers** (R2DBC, Reactive Mongo) — not JDBC.
✅ Always handle errors with `onErrorResume`.
✅ Use `flatMap` for async transformations (not `map`).
✅ Use `Schedulers.boundedElastic()` for unavoidable blocking calls.
✅ Test using **`StepVerifier`**:
```java
StepVerifier.create(service.findById("1"))
    .expectNextMatches(p -> p.getName().equals("Phone"))
    .verifyComplete();
```

---

## 🧾 21. Quick Recap

> **Spring WebFlux** = Reactive, **non-blocking**, async web framework built on **Project Reactor** (`Mono`, `Flux`) and **Netty**.
> Perfect for **high-concurrency, I/O-heavy, streaming** apps.
> Uses **backpressure** to control data flow.
> Combine reactive endpoints with **R2DBC / Reactive Mongo + WebClient** for true end-to-end reactive architecture.

---

## 📚 Typical End-to-End Reactive Stack

```
Client (Browser/Mobile)
        │
        ▼
Spring Cloud Gateway (WebFlux)
        │
        ▼
Reactive Microservices (WebFlux + WebClient)
        │
        ▼
Reactive DB (R2DBC / Mongo Reactive / Cassandra Reactive)
        │
        ▼
Kafka / RabbitMQ (Reactor Kafka)
```

---

Would you like the next set of notes on:
- 🧩 **Project Reactor deep dive (operators, hot/cold streams)**
- 🔌 **WebClient in depth (timeouts, retries, filters)**
- 💬 **WebSockets with WebFlux**
- 🧪 **Testing reactive code with `StepVerifier`**

Let me know which one to cover next! 🚀