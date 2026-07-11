# Reactive Programming with Project Reactor — Complete Guide

> Stack context: Java 21, Spring Boot / Spring WebFlux, Project Reactor, R2DBC
> This guide goes from Reactive Streams fundamentals through operators, schedulers, backpressure, sinks, context propagation, and testing — with runnable-style code examples throughout.

---

## Table of Contents

1. [Reactive Streams](#1-reactive-streams)
2. [Publishers — Mono & Flux](#2-publishers--mono--flux)
3. [Programmatically Emitting Items](#3-programmatically-emitting-items)
4. [Operators](#4-operators)
5. [Hot & Cold Publishers](#5-hot--cold-publishers)
6. [Schedulers](#6-schedulers)
7. [Backpressure / Overflow Strategy](#7-backpressure--overflow-strategy)
8. [Combining Publishers](#8-combining-publishers)
9. [Batching](#9-batching)
10. [repeat & retry](#10-repeat--retry)
11. [Sinks](#11-sinks)
12. [Context](#12-context)
13. [StepVerifier — Unit Testing](#13-stepverifier--unit-testing)

---

## 1. Reactive Streams

### 1.1 What problem does it solve?

Traditional blocking I/O ties up a thread for the duration of a request. In a Spring MVC (Servlet) app, a thread-per-request model means high concurrency requires many threads, which is expensive (context switching, memory per thread stack). Reactive Streams flips this: instead of a thread *pulling* data and blocking until it arrives, data is *pushed* to a consumer asynchronously, and a small number of threads can serve a huge number of concurrent requests non-blockingly.

Think of it the way you already think about Kafka: a **Publisher** is like a topic producer, a **Subscriber** is like a consumer group, and **backpressure** is like consumer lag control — the subscriber tells the publisher how much it can handle right now, instead of getting flooded.

### 1.2 The Reactive Streams Specification

Reactive Streams is a **specification** (JVM interfaces), not a library. Project Reactor, RxJava, Akka Streams, and Java 9's `java.util.concurrent.Flow` all implement it. It defines exactly 4 interfaces:

```java
public interface Publisher<T> {
    void subscribe(Subscriber<? super T> s);
}

public interface Subscriber<T> {
    void onSubscribe(Subscription s);
    void onNext(T t);
    void onError(Throwable t);
    void onComplete();
}

public interface Subscription {
    void request(long n);
    void cancel();
}

public interface Processor<T, R> extends Subscriber<T>, Publisher<R> {
    // acts as both — a bridge/pipeline stage
}
```

### 1.3 The protocol (rules, in plain terms)

1. A `Publisher` calls `onSubscribe` on the `Subscriber` **exactly once**, handing over a `Subscription`.
2. The `Subscriber` calls `subscription.request(n)` to say "I can handle `n` more items" — this is the backpressure signal. Nothing flows until the first `request` call.
3. The `Publisher` calls `onNext` **at most `n` times** per outstanding request — it must never send more than requested.
4. The stream terminates with exactly one of: `onComplete()` (success) or `onError()` (failure) — never both, and never followed by further `onNext`.
5. `subscription.cancel()` lets the subscriber walk away early (e.g. `take(5)` cancels after 5 items).

This request-n mechanism is the entire point of the spec: **the consumer controls the rate**, not the producer. Compare this to your Kafka mental model — a slow consumer doesn't get flooded off a topic; here, a slow subscriber simply requests less, and a well-behaved publisher (or an intermediate buffer) respects that.

### 1.4 Where Project Reactor fits

Project Reactor is Pivotal/VMware's implementation of Reactive Streams, and it's the reactive library Spring WebFlux is built on. It gives you two main Publisher implementations — `Mono` and `Flux` — plus hundreds of operators, so you rarely touch the raw `Subscriber`/`Subscription` interfaces yourself; Reactor manages the protocol correctness for you.

```
Spring MVC (Servlet stack)          Spring WebFlux (Reactive stack)
──────────────────────────          ────────────────────────────────
Thread-per-request                  Event-loop, small thread pool
Blocking JDBC                       Non-blocking R2DBC
Tomcat (default)                    Netty (default)
Controller returns T / List<T>      Controller returns Mono<T> / Flux<T>
```

---

## 2. Publishers — Mono & Flux

Reactor gives you two flavors of `Publisher<T>`:

| | `Mono<T>` | `Flux<T>` |
|---|---|---|
| Cardinality | 0 or 1 element | 0 to N (possibly infinite) elements |
| Analogy | A `CompletableFuture<T>` / `Optional<T>` | A `Stream<T>` that arrives over time |
| Typical use | Single DB row, single HTTP response, a `void` completion signal | List of rows, a stream of events, SSE, Kafka records |

### 2.1 Mono

```java
// Creating Monos
Mono<String> empty = Mono.empty();                 // completes with no value
Mono<String> justValue = Mono.just("hello");        // one value, eagerly captured
Mono<String> deferred = Mono.defer(() ->            // lazily computed per subscriber
        Mono.just(expensiveCall()));
Mono<String> fromError = Mono.error(new RuntimeException("boom"));
Mono<User> fromCallable = Mono.fromCallable(() -> userRepository.findSync(id));
Mono<User> fromFuture = Mono.fromFuture(() -> someCompletableFuture);
Mono<Void> voidSignal = Mono.fromRunnable(() -> auditLog.record("done"));

// Subscribing
justValue.subscribe(
    value -> System.out.println("Got: " + value),   // onNext
    error -> System.err.println("Error: " + error),  // onError
    () -> System.out.println("Done")                 // onComplete
);
```

**Important**: Nothing happens until `.subscribe()` is called (or Reactor's Spring adapter subscribes for you, e.g. WebFlux subscribing to your controller's returned `Mono`). Reactor publishers are **lazy** — this is a key mental shift from a Spring `@Service` method that runs immediately when called.

```java
// A service method returning a Mono — this does NOT hit the DB yet
public Mono<Order> findOrder(String id) {
    return orderRepository.findById(id)   // R2DBC repo — lazy
        .switchIfEmpty(Mono.error(new OrderNotFoundException(id)));
}
// The DB call only happens when something downstream subscribes
// (WebFlux does this automatically for controller return values)
```

### 2.2 Flux

```java
Flux<Integer> range = Flux.range(1, 5);              // 1,2,3,4,5
Flux<String> fromIterable = Flux.fromIterable(List.of("a", "b", "c"));
Flux<String> fromArray = Flux.fromArray(new String[]{"x", "y"});
Flux<Order> fromRepo = orderRepository.findAll();     // R2DBC returns Flux
Flux<Long> interval = Flux.interval(Duration.ofSeconds(1)); // 0,1,2,... every second (infinite)
Flux<String> concatValues = Flux.just("a", "b", "c");

fromIterable.subscribe(System.out::println);
```

### 2.3 Converting between Mono and Flux

```java
Mono<List<Order>> collected = fromRepo.collectList();   // Flux -> Mono<List>
Flux<Order> expanded = someMono.flux();                  // Mono -> Flux (0 or 1 element)
Mono<Order> firstOne = fromRepo.next();                   // Flux -> Mono (first element)
Mono<Order> onlyOne = fromRepo.single();                  // Flux -> Mono (errors if not exactly 1)
```

### 2.4 Real WebFlux controller example

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    private final OrderService orderService;

    @GetMapping("/{id}")
    public Mono<Order> getOrder(@PathVariable String id) {
        return orderService.findOrder(id); // WebFlux subscribes to this for you
    }

    @GetMapping
    public Flux<Order> getAllOrders() {
        return orderService.findAll();     // streamed to the client
    }

    @PostMapping
    public Mono<ResponseEntity<Order>> createOrder(@RequestBody Mono<OrderRequest> request) {
        return request
            .flatMap(orderService::create)
            .map(order -> ResponseEntity.status(HttpStatus.CREATED).body(order));
    }
}
```

---

## 3. Programmatically Emitting Items

Sometimes you're bridging a non-reactive API (a callback-based SDK, a legacy listener) into a `Flux`. Reactor gives you `create` and `generate` for this.

### 3.1 `Flux.create` — for async, multi-threaded, possibly-external event sources

`create` gives you a `FluxSink` you can push to from **any thread**, including callbacks fired by another library. It supports emitting from multiple threads (though you must serialize your own calls to it — `create` itself handles thread-safety of the sink internally).

```java
Flux<String> priceUpdates = Flux.create(sink -> {
    PriceFeedListener listener = new PriceFeedListener() {
        @Override
        public void onPriceUpdate(String symbol, double price) {
            sink.next(symbol + ":" + price);
        }
        @Override
        public void onFeedClosed() {
            sink.complete();
        }
        @Override
        public void onFeedError(Exception e) {
            sink.error(e);
        }
    };
    externalPriceFeed.subscribe(listener);
    sink.onDispose(() -> externalPriceFeed.unsubscribe(listener)); // cleanup on cancel
}, FluxSink.OverflowStrategy.BUFFER); // overflow strategy — see section 7
```

Use `create` when:
- Bridging listener/callback-based APIs (JMS, WebSocket handlers, third-party SDK callbacks)
- Emissions may come from multiple threads
- You need full control (`next`, `error`, `complete`, `onRequest`, `onCancel`, `onDispose`)

### 3.2 `Flux.generate` — for synchronous, one-at-a-time, pull-based sequences

`generate` produces items **one at a time, synchronously**, driven by downstream `request(n)` — it naturally respects backpressure since it only computes the next item when asked. Good for things like a Fibonacci sequence, a cursor-based paginated fetch, or any CPU-bound sequential computation.

```java
// Fibonacci sequence generator using mutable state
Flux<BigInteger> fibonacci = Flux.generate(
    () -> new BigInteger[]{BigInteger.ZERO, BigInteger.ONE},   // initial state
    (state, sink) -> {
        sink.next(state[0]);
        BigInteger next = state[0].add(state[1]);
        state[0] = state[1];
        state[1] = next;
        return state; // new state for next call
    }
);

fibonacci.take(10).subscribe(System.out::println);
// 0 1 1 2 3 5 8 13 21 34
```

```java
// Cursor-based pagination pulled lazily, page by page
Flux<Order> paginatedOrders = Flux.generate(
    () -> 0, // initial page number
    (page, sink) -> {
        List<Order> results = orderApi.fetchPage(page); // blocking call, wrap carefully!
        if (results.isEmpty()) {
            sink.complete();
        } else {
            results.forEach(sink::next);
        }
        return page + 1;
    }
);
```

### 3.3 `create` vs `generate` — quick comparison

| | `Flux.create` | `Flux.generate` |
|---|---|---|
| Emission style | Push (async, external) | Pull (sync, on-demand) |
| Threads | Multiple threads may call sink | Single thread, sequential |
| Backpressure | You must respect it via overflow strategy | Automatically respected |
| Typical use | Listener/callback bridging | Sequential/stateful generation |

There's also `Flux.push` — like `create` but assumes a **single producer thread** (slightly cheaper, no multi-thread serialization overhead).

---

## 4. Operators

Operators transform, filter, or react to a stream without you ever touching `Subscriber` directly. Below are the ones from your outline, grouped logically.

### 4.1 `handle` — combined filter + map, imperative-style

`handle` gives you full control per element: emit it, transform it, skip it, or terminate with an error — all via a `SynchronousSink`.

```java
Flux<String> discountCodes = Flux.just("SAVE10", "invalid", "SAVE20", "bad!!")
    .handle((code, sink) -> {
        if (code.matches("SAVE\\d+")) {
            sink.next(code.toLowerCase());   // emit transformed value
        }
        // else: silently skip (don't call sink.next => filtered out)
    });
// emits: save10, save20
```

`handle` is often the replacement for a `filter().map()` chain when the logic is more naturally expressed as "for each item, decide what (if anything) to emit."

### 4.2 Dohooks / Callbacks (side-effect operators)

These never change the data — they let you observe/react to signals for logging, metrics, cleanup.

```java
orderService.findOrder(id)
    .doOnSubscribe(sub -> log.info("Subscribed to order lookup"))
    .doOnNext(order -> log.info("Fetched order: {}", order.getId()))
    .doOnSuccess(order -> log.info("Mono completed with: {}", order)) // Mono only
    .doOnError(err -> log.error("Failed to fetch order", err))
    .doOnCancel(() -> log.warn("Subscription cancelled"))
    .doOnTerminate(() -> log.info("Terminated (success or error)"))
    .doFinally(signalType -> log.info("Finally: {}", signalType)) // ALWAYS runs — like try/finally
    .doOnComplete(() -> log.info("Flux completed")); // Flux only
```

`doFinally` is the reactive equivalent of a `finally` block — it always fires, regardless of success, error, or cancellation, making it the right place for resource cleanup (closing a connection, releasing a semaphore).

### 4.3 `limitRate` — throttling request-n

Controls how many items are requested from upstream in each batch, useful for controlling memory pressure or downstream load without dropping data (unlike overflow strategies, which are about handling **overflow**, `limitRate` shapes the demand itself).

```java
Flux.range(1, 1000)
    .limitRate(50) // requests upstream in chunks of 50 instead of Long.MAX_VALUE
    .subscribe(this::processItem);
```

### 4.4 `delay` family

```java
Mono.just("hello").delayElement(Duration.ofSeconds(2)); // delays emission
Flux.just(1, 2, 3).delaySequence(Duration.ofSeconds(1)); // delays entire sequence start
Mono.just("x").delaySubscription(Duration.ofSeconds(5)); // delays the subscribe() call itself
```

### 4.5 `timeout`

Errors out (`TimeoutException` by default) if no item arrives within the given duration — critical for calling downstream services (analogous to a WebClient response timeout, or a Resilience4j `TimeLimiter`).

```java
webClient.get().uri("/inventory/{sku}", sku)
    .retrieve()
    .bodyToMono(Inventory.class)
    .timeout(Duration.ofSeconds(3))
    .onErrorResume(TimeoutException.class,
        e -> Mono.just(Inventory.unavailable(sku))); // fallback on timeout
```

You can also supply a fallback publisher directly: `.timeout(Duration.ofSeconds(3), fallbackMono)`.

### 4.6 on-error operators

This is the reactive equivalent of `try/catch` blocks, and there are several distinct tools:

```java
// onErrorReturn — swallow the error, emit a static fallback value
priceService.getPrice(sku)
    .onErrorReturn(Price.ZERO);

// onErrorReturn with predicate — only for specific exceptions
priceService.getPrice(sku)
    .onErrorReturn(PriceNotFoundException.class, Price.ZERO);

// onErrorResume — swallow the error, switch to a fallback PUBLISHER (can be async)
priceService.getPrice(sku)
    .onErrorResume(PriceServiceException.class,
        ex -> cacheService.getCachedPrice(sku));

// onErrorContinue — skip the FAILING ELEMENT and continue the sequence (Flux only, use carefully)
Flux.just("1", "2", "abc", "4")
    .map(Integer::parseInt)
    .onErrorContinue((ex, val) -> log.warn("Skipping bad value: {}", val));
// emits: 1, 2, 4  (skips "abc")

// onErrorMap — rethrow as a different, more meaningful exception type
orderRepository.save(order)
    .onErrorMap(DataIntegrityViolationException.class,
        ex -> new DuplicateOrderException(order.getId(), ex));

// doOnError — side-effect only (logging), doesn't handle/recover the error, error still propagates
orderRepository.save(order)
    .doOnError(ex -> log.error("Save failed for order {}", order.getId(), ex));
```

**Rule of thumb**: `doOnError` for logging only (error still propagates); `onErrorReturn`/`onErrorResume` for recovery; `onErrorMap` for translating exception types (e.g., turning a low-level R2DBC exception into a domain exception, the same way you'd translate exceptions at a service boundary in a layered Spring app); `onErrorContinue` for skip-and-keep-going on a per-element basis (use sparingly — it can silently swallow issues if operators upstream don't support it properly).

### 4.7 `transform` vs `transformDeferred`

Both let you extract and reuse a chain of operators — like a reusable "operator pipeline" you'd factor out into a shared library (similar to how you built shared libraries on the Carlsberg BFF project).

```java
Function<Flux<Order>, Flux<Order>> applyBusinessRules = flux -> flux
    .filter(o -> o.getStatus() != OrderStatus.CANCELLED)
    .doOnNext(o -> metrics.increment("orders.processed"));

// transform: the function is applied ONCE at assembly time — same pipeline for every subscriber
orderFlux.transform(applyBusinessRules);

// transformDeferred: the function is invoked PER SUBSCRIBER — allows per-subscription state
orderFlux.transformDeferred(flux -> {
    String requestId = UUID.randomUUID().toString(); // fresh per subscriber
    return flux.doOnNext(o -> log.info("[{}] processing {}", requestId, o.getId()));
});
```

### 4.8 `switchIfEmpty` / `defaultIfEmpty`

Both handle the "no data" case, but differently:

```java
// defaultIfEmpty — supply a static fallback VALUE
userRepository.findById(id)
    .defaultIfEmpty(User.guest());

// switchIfEmpty — switch to a fallback PUBLISHER (can be async, e.g. a lookup elsewhere)
userRepository.findById(id)
    .switchIfEmpty(externalUserService.fetchUser(id))  // try another source
    .switchIfEmpty(Mono.error(new UserNotFoundException(id))); // then error if still empty
```

This chained `switchIfEmpty` pattern is a common way to express "try cache, then DB, then remote service, then fail" pipelines.

---

## 5. Hot & Cold Publishers

### 5.1 Cold publishers (the default)

A **cold** publisher replays its whole sequence for **every** new subscriber, independently. Think of it like a Kafka topic being re-read from offset 0 for each new consumer group — each subscriber gets its own independent execution.

```java
Flux<Integer> cold = Flux.range(1, 3)
    .doOnSubscribe(s -> System.out.println("New subscription started"));

cold.subscribe(v -> System.out.println("A: " + v));
cold.subscribe(v -> System.out.println("B: " + v));
// Output:
// New subscription started
// A: 1
// A: 2
// A: 3
// New subscription started
// B: 1
// B: 2
// B: 3
```
Most publishers you build (DB queries, HTTP calls) are cold by default — each subscription re-executes the underlying work (e.g. re-runs the SQL query).

### 5.2 Hot publishers

A **hot** publisher emits regardless of whether anyone is subscribed, and subscribers only see items emitted **after** they subscribe — like tuning into a live radio broadcast or a live Kafka topic where you only see new messages from the moment you subscribe (no replay of history unless you specifically opt into replay).

### 5.3 `share()` — turn a cold Flux into a hot one (multicast + auto-connect)

`share()` = `publish().refCount(1)` — the source is subscribed to once, shared among all subscribers, and the source connects when the first subscriber arrives and disconnects when the last one leaves.

```java
Flux<Long> ticker = Flux.interval(Duration.ofSeconds(1))
    .doOnSubscribe(s -> System.out.println("Connecting to source"))
    .share();

ticker.subscribe(v -> System.out.println("Sub1: " + v));
Thread.sleep(2000);
ticker.subscribe(v -> System.out.println("Sub2: " + v)); // joins mid-stream, misses 0 and 1
```
Use `share()` when multiple subscribers should observe the **same live execution** — e.g. broadcasting price ticks to multiple WebSocket sessions without re-running the underlying feed subscription per client.

### 5.4 `publish().autoConnect(n)`

`autoConnect(n)` connects to the source once **n subscribers** have subscribed — but unlike `refCount`, once connected it **stays connected** even if subscribers later leave (no automatic disconnect/reconnect cycling).

```java
ConnectableFlux<Long> connectable = Flux.interval(Duration.ofSeconds(1)).publish();

Flux<Long> autoConnecting = connectable.autoConnect(2); // waits for 2 subscribers

autoConnecting.subscribe(v -> System.out.println("Sub1: " + v)); // not connected yet
autoConnecting.subscribe(v -> System.out.println("Sub2: " + v)); // NOW it connects
```

Related: `connect()` on a `ConnectableFlux` lets you manually decide exactly when the source starts, independent of any subscriber count — full manual control, similar to explicitly starting a Kafka producer only when your app is ready.

### 5.5 `cache()` — replay past emissions to late subscribers

`cache()` makes a cold publisher hot **and** remembers past emissions, replaying them to any subscriber that joins late — like a Kafka topic with infinite retention that new consumers can always read from the beginning.

```java
Mono<Config> cachedConfig = configService.loadConfig() // expensive remote call
    .cache(); // subscribe once, cache the result forever

cachedConfig.subscribe(c -> System.out.println("First: " + c));  // triggers the actual call
cachedConfig.subscribe(c -> System.out.println("Second: " + c)); // gets cached value, no new call
```

```java
// Time-bound and size-bound caching variants
someFlux.cache(Duration.ofMinutes(5));              // expires after 5 minutes
someFlux.cache(100);                                  // keeps last 100 items only
someFlux.cache(100, Duration.ofMinutes(5));          // both bounds
```

This is a very common pattern for caching a slowly-changing config or reference-data lookup (e.g. a feature-flag Mono) without needing a separate cache layer like Redis for cheap, in-memory, per-instance caching.

### 5.6 Summary table

| Operator | Connects when | Late subscriber sees | Disconnects when |
|---|---|---|---|
| (cold, default) | Every `subscribe()` | Full sequence, independently | N/A — each has its own run |
| `share()` | First subscriber | Only future emissions | Last subscriber leaves (then resets) |
| `publish().autoConnect(n)` | n-th subscriber | Only future emissions | Never automatically |
| `cache()` | First subscriber | **Replayed** past emissions | Never (unless bounded) |

---

## 6. Schedulers

Reactor is single-threaded by default — everything runs on whatever thread called `subscribe()`, unless you explicitly introduce a `Scheduler`. This is a deliberate design: **you** decide where work happens, rather than the framework guessing.

### 6.1 Available Scheduler types

```java
Schedulers.immediate();      // no thread switch — runs on the current thread
Schedulers.single();          // one single, reused daemon thread — for sequential work
Schedulers.boundedElastic();  // bounded pool of on-demand threads — for blocking I/O calls
Schedulers.parallel();        // fixed pool sized to CPU cores — for CPU-bound work
Schedulers.newParallel("custom-pool", 4); // custom named parallel pool
```

Rule of thumb mapping to Spring-world instincts:
- `parallel()` ≈ a CPU-bound thread pool sized to cores — for computation-heavy transforms
- `boundedElastic()` ≈ your JDBC-blocking-call escape hatch — use this to wrap any **blocking** call (legacy JDBC, blocking SDK, file I/O) so it doesn't starve the small Netty event-loop threads
- `immediate()` ≈ "don't switch threads" — the default

### 6.2 `publishOn` vs `subscribeOn`

This trips up almost everyone at first — think of the reactive chain as a pipeline being built top-to-bottom, but **executed bottom-to-top** when a subscription happens (the `subscribe()` signal flows upstream, then data flows back downstream).

- **`subscribeOn`**: affects where the **subscription (and the source emission)** happens. Position in the chain **doesn't matter** — it affects the whole chain's origin. Only the *first* `subscribeOn` in a chain has effect.
- **`publishOn`**: affects where execution happens **from that point downward** in the chain. Position matters — you can have multiple `publishOn` calls to switch threads at different stages.

```java
Flux.range(1, 3)
    .doOnNext(i -> System.out.println("map1 on " + Thread.currentThread().getName()))
    .map(i -> i * 2)
    .publishOn(Schedulers.parallel())          // switch thread HERE onward
    .doOnNext(i -> System.out.println("map2 on " + Thread.currentThread().getName()))
    .map(i -> i + 1)
    .subscribeOn(Schedulers.boundedElastic())  // affects the SOURCE, regardless of position
    .subscribe();
```

Practical example — wrapping a blocking legacy call inside an otherwise reactive Spring WebFlux pipeline:

```java
public Mono<LegacyResponse> callLegacyBlockingSystem(Request req) {
    return Mono.fromCallable(() -> legacyBlockingClient.call(req)) // blocking!
        .subscribeOn(Schedulers.boundedElastic()) // run the blocking call off the event loop
        .timeout(Duration.ofSeconds(5));
}
```

**Never block a Netty event-loop thread.** If you must call something blocking (a synchronous JDBC driver, a blocking SDK), always wrap it with `Mono.fromCallable(...).subscribeOn(Schedulers.boundedElastic())` — this is directly analogous to your AKS/Eureka debugging instinct of isolating blocking dependencies so they don't cascade-fail the whole pool.

---

## 7. Backpressure / Overflow Strategy

Backpressure (section 1.3) is the **protocol**-level mechanism (`request(n)`). But sometimes a source genuinely can't be slowed down — e.g. `Flux.interval()` ticking every millisecond, or `Flux.create` wrapping an external push-based feed that has no "pause" button. **Overflow strategies** define what happens when items arrive faster than they're requested.

```java
Flux.create(sink -> {
    // fast external producer, no way to signal "slow down" to it
    fastFeed.onMessage(sink::next);
}, FluxSink.OverflowStrategy.BUFFER); // or ERROR, DROP, LATEST, IGNORE
```

| Strategy | Behavior |
|---|---|
| `ERROR` | Throws `IllegalStateException` (`Overflow`) if downstream can't keep up — fail fast |
| `DROP` | Silently drops the **newest** items that don't fit (once buffer/demand is exhausted) |
| `LATEST` | Keeps only the **most recent** item, discarding older unconsumed ones — good for "current state" streams like a live price ticker where only the latest value matters |
| `BUFFER` | Buffers unboundedly (risk of `OutOfMemoryError` under sustained overflow) — or bounded with an eviction policy |
| `IGNORE` | Does nothing — completely ignores backpressure requests, letting `onNext` calls pass straight through (dangerous — use only if you know the downstream truly can't be overwhelmed) |

### Bounded `BUFFER` with eviction

```java
Flux.create(sink -> fastFeed.onMessage(sink::next),
        FluxSink.OverflowStrategy.BUFFER)
    .onBackpressureBuffer(1000,
        dropped -> log.warn("Dropped due to full buffer: {}", dropped),
        BufferOverflowStrategy.DROP_OLDEST);
```

### Standalone `onBackpressureX` operators

These can be applied to any `Flux`, not just `create`-based ones:

```java
fastSource.onBackpressureDrop(item -> log.warn("Dropped: {}", item));
fastSource.onBackpressureLatest();
fastSource.onBackpressureBuffer(500); // bounded buffer, errors when full by default
fastSource.onBackpressureError();     // explicit fail-fast
```

**Choosing a strategy**, using your reactive-price-feed / order-service mental model:
- **`LATEST`** — live dashboards, current inventory count, GPS position (only the newest value matters)
- **`DROP`** — metrics/telemetry pings where losing a few samples is fine
- **`BUFFER`** (bounded) — order events, financial transactions where you can't lose data but can tolerate some delay
- **`ERROR`** — when overflow indicates a bug or a genuinely broken invariant you want to surface immediately

---

## 8. Combining Publishers

### 8.1 `flatMap` — async, concurrent, unordered merge

Maps each element to a new `Publisher` and **merges** all resulting publishers, running them **concurrently** (default concurrency: 256). Order of results is **not guaranteed** to match the source order.

```java
Flux<String> orderIds = Flux.just("o1", "o2", "o3");

Flux<OrderDetail> details = orderIds
    .flatMap(id -> orderService.fetchDetail(id)); // fires all 3 calls concurrently

// Bounded concurrency — like limiting a thread pool size
Flux<OrderDetail> boundedDetails = orderIds
    .flatMap(id -> orderService.fetchDetail(id), 5); // max 5 in-flight subscriptions at once
```
Use `flatMap` for **independent** async calls where order doesn't matter and you want maximum throughput — e.g. enriching a list of order IDs by calling a downstream price/inventory service for each, in parallel (the classic BFF fan-out pattern you've built before).

### 8.2 `concatMap` — sequential, ordered merge

Like `flatMap` but processes one inner publisher **fully** before starting the next — preserves order, no concurrency.

```java
Flux<OrderDetail> orderedDetails = orderIds
    .concatMap(id -> orderService.fetchDetail(id)); // strictly sequential, order preserved
```

### 8.3 `merge` — combine multiple existing publishers, interleaved by arrival time

```java
Flux<String> feed1 = Flux.interval(Duration.ofMillis(100)).map(i -> "feed1-" + i);
Flux<String> feed2 = Flux.interval(Duration.ofMillis(150)).map(i -> "feed2-" + i);

Flux<String> merged = Flux.merge(feed1, feed2); // interleaved as items arrive, no fixed order
```
`merge` subscribes to all sources **eagerly and concurrently** — good for combining multiple live/hot sources (e.g. merging order events and inventory events into a single audit stream).

### 8.4 `zip` — pairwise combine, waits for one from each source

```java
Mono<Order> orderMono = orderRepository.findById(orderId);
Mono<Customer> customerMono = customerRepository.findById(customerId);
Mono<Inventory> inventoryMono = inventoryService.check(sku);

Mono<OrderSummary> summary = Mono.zip(orderMono, customerMono, inventoryMono)
    .map(tuple -> new OrderSummary(tuple.getT1(), tuple.getT2(), tuple.getT3()));

// Flux.zip pairs elements by INDEX — waits for the slowest source per pair
Flux<String> names = Flux.just("Alice", "Bob");
Flux<Integer> ages  = Flux.just(30, 25);
Flux<String> zipped = Flux.zip(names, ages, (n, a) -> n + " is " + a); // "Alice is 30", "Bob is 25"
```
Use `zip` when you need **all** of several independent results combined together — this is your BFF aggregation pattern (order + price + inventory in one response) done in parallel with a single wait point, instead of chaining sequential `flatMap` calls.

### 8.5 `concat` — sequential, one publisher fully before the next, order strictly preserved by source

```java
Flux<String> combined = Flux.concat(
    Flux.just("first-batch-a", "first-batch-b"),
    Flux.just("second-batch-a", "second-batch-b")
);
// always: first-batch-a, first-batch-b, second-batch-a, second-batch-b — regardless of timing
```
Difference from `merge`: `concat` subscribes to the second source only **after** the first completes — fully deterministic ordering, no interleaving, no concurrency.

### 8.6 Quick decision guide

| Need | Operator |
|---|---|
| Map each item to an async call, don't care about order, want concurrency | `flatMap` |
| Map each item to an async call, order matters, sequential is fine | `concatMap` |
| Combine existing hot/live sources, interleaved | `merge` |
| Combine existing sources, strict sequence, one after another | `concat` |
| Wait for one value from each of several sources and combine them | `zip` |

---

## 9. Batching

### 9.1 `buffer` — collect N items (or by time/predicate) into a `List`

```java
Flux.range(1, 10).buffer(3).subscribe(System.out::println);
// [1,2,3] [4,5,6] [7,8,9] [10]

// Time-based buffering — batch DB writes every 500ms
orderEventFlux.buffer(Duration.ofMillis(500))
    .filter(batch -> !batch.isEmpty())
    .flatMap(batch -> orderRepository.saveAll(batch).then());

// Size OR time, whichever comes first — classic batching pattern
eventFlux.bufferTimeout(100, Duration.ofSeconds(1))
    .flatMap(batch -> bulkInsert(batch));

// Predicate-based buffering — start a new buffer when predicate is true
Flux.just(1, 2, 3, 100, 4, 5, 200, 6)
    .bufferUntil(i -> i > 50); // [1,2,3,100] [4,5,200] [6]
```
`bufferTimeout` is exactly the pattern you'd reach for to batch Kafka-consumed events into bulk R2DBC inserts — trading a little latency for much higher DB write throughput.

### 9.2 `window` — like `buffer`, but produces `Flux<Flux<T>>` instead of `Flux<List<T>>`

Use `window` instead of `buffer` when the batches themselves are large or unbounded (so you don't materialize a huge `List` in memory) — each window is itself a reactive `Flux` you can process with further operators/backpressure, rather than a fully realized list.

```java
Flux.range(1, 10)
    .window(3)
    .flatMap(windowFlux -> windowFlux.collectList()) // process each window as a sub-stream
    .subscribe(System.out::println);
// [1,2,3] [4,5,6] [7,8,9] [10]

// Windowing by time — useful for rolling metrics/aggregation windows
metricsFlux.window(Duration.ofSeconds(10))
    .flatMap(window -> window.count())
    .subscribe(count -> log.info("Events in last window: {}", count));
```

### 9.3 `groupBy` — partition a stream into sub-streams by key

Produces a `Flux<GroupedFlux<K, T>>` — think of it as an in-memory equivalent of Kafka partitioning by key, splitting one stream into many keyed sub-streams **within the same process**.

```java
Flux<Order> orders = orderRepository.findAll();

orders.groupBy(Order::getStatus)
    .flatMap(groupedFlux -> groupedFlux
        .collectList()
        .map(list -> Map.entry(groupedFlux.key(), list)))
    .subscribe(entry -> log.info("Status {}: {} orders", entry.getKey(), entry.getValue().size()));
```

**Caution**: each `GroupedFlux` must be subscribed to (consumed) or it will build up in memory — don't `groupBy` and then ignore some groups.

### 9.4 buffer vs window vs groupBy

| | Splits by | Result | Use when |
|---|---|---|---|
| `buffer` | count / time / predicate | `Flux<List<T>>` | You want a materialized batch, e.g. for bulk-save |
| `window` | count / time / predicate | `Flux<Flux<T>>` | Batches are large or you want further reactive processing per batch |
| `groupBy` | a key function | `Flux<GroupedFlux<K,T>>` | You need to partition by value, not by arrival order |

---

## 10. `repeat` & `retry`

Both re-subscribe to the source, but for opposite reasons: `repeat` re-runs a **successfully completed** sequence; `retry` re-runs a sequence that **errored**.

### 10.1 `repeat`

```java
// Poll a status endpoint every time the previous check completes, up to 5 times
Mono<Status> pollStatus = statusService.checkStatus(jobId);

pollStatus.repeat(5).subscribe(status -> log.info("Status: {}", status));

// repeat with a predicate — keep repeating while condition holds
pollStatus.repeat(status -> !status.isComplete())
    .subscribe(status -> log.info("Polling... {}", status));
```

### 10.2 `retry`

```java
// Simple retry — re-subscribe up to 3 times on ANY error
webClient.get().uri("/inventory")
    .retrieve()
    .bodyToMono(Inventory.class)
    .retry(3);

// retryWhen with Reactor's built-in retry specs — exponential backoff (very common in Spring WebFlux clients)
webClient.get().uri("/inventory")
    .retrieve()
    .bodyToMono(Inventory.class)
    .retryWhen(Retry.backoff(3, Duration.ofMillis(200))
        .maxBackoff(Duration.ofSeconds(2))
        .jitter(0.5)
        .filter(ex -> ex instanceof WebClientResponseException.ServiceUnavailable)
        .onRetryExhaustedThrow((spec, signal) ->
            new ServiceUnavailableException("Inventory service unavailable after retries", signal.failure())));

// Fixed-delay retry
someMono.retryWhen(Retry.fixedDelay(5, Duration.ofSeconds(1)));
```

This maps directly onto the Resilience4j retry config you've worked with — `Retry.backoff` is Reactor's native equivalent of a resilience4j retry policy, and you can combine `.timeout()` + `.retryWhen()` + `.onErrorResume()` in a single WebClient chain to get a full circuit-breaker-adjacent resilience pipeline without an extra library.

**Common bug to watch for**: putting `.retry()` *before* an operator that has side effects (like `doOnNext` calling an external system) will re-trigger those side effects on every retry — place side-effecting operators carefully relative to where the retry boundary is.

---

## 11. Sinks

`Sinks` (Reactor 3.4+) is the modern, safe replacement for manually implementing `Processor` or using the older `EmitterProcessor`/`ReplayProcessor` classes directly. A `Sink` lets **imperative code** push values into a reactive stream programmatically — the bridge point between your non-reactive code (e.g. an event handler, a scheduled job) and a reactive pipeline.

### 11.1 `Sinks.one()` — a programmatically-completable Mono

```java
Sinks.One<String> sink = Sinks.one();

// Somewhere else in imperative code:
sink.tryEmitValue("hello");        // or sink.tryEmitError(ex), sink.tryEmitEmpty()

Mono<String> mono = sink.asMono();
mono.subscribe(System.out::println); // "hello"
```
Good for bridging a single callback result (e.g. a one-shot async SDK callback) into a `Mono` your Spring WebFlux layer can return.

### 11.2 `Sinks.many().unicast()` — exactly one subscriber

```java
Sinks.Many<String> unicastSink = Sinks.many().unicast().onBackpressureBuffer();

unicastSink.tryEmitNext("event-1");
unicastSink.tryEmitNext("event-2");

Flux<String> flux = unicastSink.asFlux();
flux.subscribe(System.out::println); // works — first subscriber

// A SECOND subscribe() here would throw IllegalStateException — unicast allows only ONE subscriber ever
```
Use when there is genuinely only ever one consumer — e.g. a single SSE connection tied 1:1 to a sink instance created per-request.

### 11.3 `Sinks.many().multicast()` — multiple subscribers, but only future emissions

```java
Sinks.Many<String> multicastSink = Sinks.many().multicast().onBackpressureBuffer();

Flux<String> flux = multicastSink.asFlux();
flux.subscribe(v -> System.out.println("Sub1: " + v));

multicastSink.tryEmitNext("event-1"); // Sub1 receives it

flux.subscribe(v -> System.out.println("Sub2: " + v)); // joins late
multicastSink.tryEmitNext("event-2"); // BOTH Sub1 and Sub2 receive this one
```
This is your fan-out broadcast primitive — e.g. broadcasting live order-status changes to every currently-connected WebSocket/SSE subscriber, similar in spirit to a hot Kafka-consumer-group broadcast, but in-process.

### 11.4 `Sinks.many().replay()` — multiple subscribers, with history replay

```java
Sinks.Many<String> replaySink = Sinks.many().replay().limit(10); // keep last 10 events

replaySink.tryEmitNext("event-1");
replaySink.tryEmitNext("event-2");

Flux<String> flux = replaySink.asFlux();
flux.subscribe(System.out::println); // even though it subscribed late, it gets event-1, event-2 replayed

// Variants:
Sinks.many().replay().all();            // unbounded replay history
Sinks.many().replay().limit(50);        // last 50 items
Sinks.many().replay().limit(Duration.ofMinutes(5)); // time-bounded replay
```

### 11.5 Emission methods — `tryEmitX` vs `emitX`

```java
// tryEmitNext returns a Sinks.EmitResult you should check/handle — non-throwing
Sinks.EmitResult result = sink.tryEmitNext("value");
if (result.isFailure()) {
    log.warn("Emission failed: {}", result);
}

// emitNext takes a failure handler — throws by default unless you supply retry logic
sink.emitNext("value", Sinks.EmitFailureHandler.FAIL_FAST);
```
Reactor deliberately makes you **handle emission failures explicitly** (buffer full, no subscribers, terminated sink) rather than silently swallowing them — treat `tryEmitNext`'s result the way you'd treat a Kafka producer's send callback/future: don't ignore it in production code.

### 11.6 Summary table

| Sink | Cardinality | Subscribers | Late subscriber sees |
|---|---|---|---|
| `Sinks.one()` | 0 or 1 value | Any number (Mono semantics — all get the same single value) | The eventual single value |
| `Sinks.many().unicast()` | N values | Exactly 1 | N/A — only one allowed |
| `Sinks.many().multicast()` | N values | Many | Only future emissions |
| `Sinks.many().replay()` | N values | Many | Replayed history (bounded or unbounded) |

---

## 12. Context

Reactor `Context` is a **per-subscription**, immutable, key-value store attached to the reactive chain — the reactive answer to `ThreadLocal`, which doesn't work reliably in reactive code because a single logical request can hop across multiple threads (`publishOn`/`subscribeOn` switches).

### 12.1 Why not `ThreadLocal`?

```java
// BROKEN in reactive code — ThreadLocal is tied to a thread, but reactive execution
// can move a logical operation across several different threads mid-chain.
private static final ThreadLocal<String> requestId = new ThreadLocal<>();
```
Since a `Mono`/`Flux` chain can execute its stages on different threads via `publishOn`, a value set in a `ThreadLocal` upstream may simply not be visible downstream. `Context` solves this by traveling **with the subscription itself**, independent of which thread executes each stage.

### 12.2 Writing and reading Context

```java
Mono<String> withContext = Mono.deferContextual(ctx ->
        Mono.just("Processing for user: " + ctx.get("userId")))
    .contextWrite(Context.of("userId", "u-123"));

withContext.subscribe(System.out::println); // "Processing for user: u-123"
```

**Critical gotcha**: `Context` flows **top-down is wrong — it actually propagates upstream-to-downstream in terms of visibility, but is written bottom-up in the chain.** In practice: `contextWrite` makes the value visible to operators **above** it (earlier in the chain, i.e. upstream), because Reactor propagates context during the *subscribe* phase, which travels from downstream to upstream. So `contextWrite` should generally be placed **at the end** of your chain (or as close to `.subscribe()` as possible) for it to be visible to operators higher up.

```java
Mono<String> mono = Mono.deferContextual(ctx -> Mono.just("value=" + ctx.get("key")))
    .map(s -> s + " (mapped)")
    .contextWrite(Context.of("key", "42")); // written LAST — visible to everything above it

mono.subscribe(System.out::println); // "value=42 (mapped)"
```

### 12.3 Practical Spring WebFlux use case: propagating a correlation/trace ID

```java
@RestController
public class OrderController {

    @GetMapping("/orders/{id}")
    public Mono<Order> getOrder(@PathVariable String id,
                                 @RequestHeader("X-Correlation-Id") String correlationId) {
        return orderService.findOrder(id)
            .contextWrite(Context.of("correlationId", correlationId));
    }
}

@Service
public class OrderService {
    public Mono<Order> findOrder(String id) {
        return orderRepository.findById(id)
            .flatMap(order -> Mono.deferContextual(ctx -> {
                String correlationId = ctx.getOrDefault("correlationId", "unknown");
                log.info("[{}] Found order {}", correlationId, id);
                return Mono.just(order);
            }));
    }
}
```
This is the reactive equivalent of an MDC-based correlation ID you'd normally thread through a blocking Spring MVC filter chain — Spring's `ContextView`/MDC bridge utilities (or Micrometer's context propagation library) can also sync Reactor `Context` back into MDC for log correlation across async boundaries, which matters a lot for your AKS multi-service tracing.

### 12.4 Merging context

```java
someFlux
    .contextWrite(Context.of("a", "1"))
    .contextWrite(ctx -> ctx.put("b", "2")); // merges with existing context, doesn't replace it
```

---

## 13. StepVerifier — Unit Testing

`StepVerifier` (from `reactor-test`) lets you assert on a reactive sequence step-by-step — the reactive equivalent of asserting against a mocked `CompletableFuture` or a blocking call's return value, but for async streams.

### 13.1 Dependency

```xml
<dependency>
    <groupId>io.projectreactor</groupId>
    <artifactId>reactor-test</artifactId>
    <scope>test</scope>
</dependency>
```

### 13.2 Basic assertions

```java
@Test
void shouldEmitExpectedValues() {
    Flux<Integer> flux = Flux.just(1, 2, 3);

    StepVerifier.create(flux)
        .expectNext(1)
        .expectNext(2, 3)      // can assert multiple at once too
        .verifyComplete();      // asserts onComplete was called — always terminate with verify*()
}

@Test
void shouldPropagateError() {
    Mono<String> mono = Mono.error(new IllegalStateException("boom"));

    StepVerifier.create(mono)
        .expectErrorMessage("boom")
        .verify();
}

@Test
void shouldMatchExactSequence() {
    StepVerifier.create(Flux.just("a", "b", "c"))
        .expectNextSequence(List.of("a", "b", "c"))
        .verifyComplete();
}
```

### 13.3 Testing a service that depends on a repository (mock + StepVerifier)

```java
@Test
void shouldReturnOrderWhenFound() {
    OrderRepository repo = mock(OrderRepository.class);
    when(repo.findById("o1")).thenReturn(Mono.just(new Order("o1", "PLACED")));

    OrderService service = new OrderService(repo);

    StepVerifier.create(service.findOrder("o1"))
        .expectNextMatches(order -> order.getId().equals("o1") && order.getStatus().equals("PLACED"))
        .verifyComplete();
}

@Test
void shouldErrorWhenOrderNotFound() {
    OrderRepository repo = mock(OrderRepository.class);
    when(repo.findById("missing")).thenReturn(Mono.empty());

    OrderService service = new OrderService(repo);

    StepVerifier.create(service.findOrder("missing"))
        .expectErrorMatches(ex -> ex instanceof OrderNotFoundException)
        .verify();
}
```

### 13.4 Testing time-based streams with a `VirtualTimeScheduler`

Instead of your test actually sleeping for real seconds, `withVirtualTime` lets Reactor **simulate elapsed time instantly** — essential for testing `interval`, `delayElement`, `timeout`, and retry-backoff logic without slow tests.

```java
@Test
void shouldEmitEveryHourWithoutWaitingRealTime() {
    StepVerifier.withVirtualTime(() -> Flux.interval(Duration.ofHours(1)).take(3))
        .expectSubscription()
        .expectNoEvent(Duration.ofHours(1))   // assert nothing happens before the first tick
        .expectNext(0L)
        .thenAwait(Duration.ofHours(1))
        .expectNext(1L)
        .thenAwait(Duration.ofHours(1))
        .expectNext(2L)
        .verifyComplete();
}
```

### 13.5 Backpressure testing

```java
@Test
void shouldRespectRequestedDemand() {
    StepVerifier.create(Flux.range(1, 10), 2) // start by requesting only 2 items
        .expectNext(1, 2)
        .thenRequest(3)                        // request 3 more
        .expectNext(3, 4, 5)
        .thenCancel()                           // cancel — remaining items never arrive
        .verify();
}
```

### 13.6 Asserting Context propagation

```java
@Test
void shouldPropagateContext() {
    Mono<String> mono = Mono.deferContextual(ctx -> Mono.just(ctx.get("userId")));

    StepVerifier.create(mono, StepVerifierOptions.create()
            .withInitialContext(Context.of("userId", "u-123")))
        .expectNext("u-123")
        .verifyComplete();
}
```

### 13.7 Useful assertion methods, quick reference

| Method | Purpose |
|---|---|
| `expectNext(T...)` | Assert exact next value(s) |
| `expectNextMatches(Predicate)` | Assert next value against a predicate |
| `expectNextCount(long)` | Assert N values arrive, without inspecting them |
| `assertNext(Consumer)` | Run arbitrary assertions (e.g. AssertJ) on the next value |
| `expectComplete()` / `verifyComplete()` | Assert successful completion |
| `expectError()` / `expectErrorMatches()` / `expectErrorMessage()` | Assert error termination |
| `expectTimeout(Duration)` | Assert the sequence does NOT complete within a duration |
| `thenAwait(Duration)` | Advance virtual time (with `withVirtualTime`) |
| `thenCancel()` | Cancel the subscription mid-test |
| `verifyTimeout(Duration)` | Verify overall test doesn't exceed a real wall-clock duration |

---

## Closing notes — how these pieces fit together

For your health-records platform (Java 21, Spring WebFlux, R2DBC, Kafka, AKS), a realistic end-to-end flow uses nearly every concept above:

1. A controller returns a `Mono`/`Flux` (§2) built from an R2DBC repository call.
2. Business logic composes results from multiple sources using `zip`/`flatMap` (§8) — e.g. combining a patient record with an access-grant check.
3. Blocking legacy calls (if any) are isolated with `subscribeOn(Schedulers.boundedElastic())` (§6).
4. External event streams (Kafka consumer records, SSE to a doctor's dashboard) are bridged with `Flux.create` (§3) or `Sinks.many()` (§11), and shared across multiple listeners with `share()` (§5).
5. Resilience against downstream failures is handled with `timeout` + `retryWhen` + `onErrorResume` (§4, §10).
6. High-volume audit/event writes are batched with `bufferTimeout` before a bulk R2DBC insert (§9).
7. A correlation/trace ID for each request flows through the whole chain via `Context` (§12), independent of which thread executes each stage.
8. Every piece of this is verified with `StepVerifier`, including simulated time for anything using `delay`/`interval`/`retry` backoff (§13).

Master each section in isolation first using the code snippets above, then revisit this closing flow — it should read like a description of a system you could actually build.