<p><a target="_blank" href="https://app.eraser.io/workspace/bciRDxY6PbaqC6harEo4" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>

# Performance & Optimization — Complete Guide
A practical guide to the techniques that turn a "correct" backend into a **fast, scalable, production-grade** one.

---

# Part 1: The Mental Model
Performance optimization is mostly about **managing scarce resources** — threads, sockets, memory, CPU, network bandwidth — and **avoiding waste**:

- **Don't create what you can reuse** → pooling.
- **Don't block when you can yield** → async / non-blocking.
- **Don't transfer one item when you can transfer many** → batching.
- **Don't transfer big data when you can transfer small** → compression, pagination.
- **Don't push faster than the consumer can drink** → backpressure.
- **Don't load everything into memory when you can stream** → streaming.
These ten patterns are different angles on those principles.

---

# Part 2: Connection Pooling
## 2.1 What It Is
A **cache of pre-established connections** (DB, HTTP, gRPC, Redis, AMQP) that are reused across requests instead of opening and closing one per request.

```
Without pooling:
     request → open conn → query → close conn   (~1–50 ms overhead per request)
With pooling:
     request → borrow conn → query → return conn   (microseconds)
```
## 2.2 Why It's Critical
Opening a TCP connection costs:

- **3-way TCP handshake** (1 RTT)
- **TLS handshake** (1–2 RTT, plus crypto)
- **Authentication** (DB login, etc.)
For an internal database, that easily runs **20–100 ms**, dominating any sub-ms query.

Beyond latency:

- **TCP slow start** — fresh connections are slow even after they're open.
- **Port exhaustion** — Linux has ~28 K ephemeral ports; high churn runs out.
- **DB server exhaustion** — Postgres tops out around a few hundred connections; without pooling each app instance would need 10× more.
## 2.3 Key Knobs
- **Min idle / Max size** — keep some warm; cap to prevent overload.
- **Max wait time** — fail fast when pool exhausted.
- **Idle timeout** — close idle connections to free server resources.
- **Validation query** — detect dead connections (`SELECT 1` ).
- **Leak detection** — alert if a connection isn't returned.
## 2.4 Tools
- **JDBC**: HikariCP (the gold standard), C3P0, Tomcat JDBC.
- **Postgres**: PgBouncer (proxy-side pooling — essential at scale).
- **HTTP**: Apache HttpClient, OkHttp, .NET `HttpClientFactory` , Go default `http.Client` .
- **Connection multiplexing in HTTP/2 and gRPC** — one TCP connection carries many concurrent streams; reduces (but doesn't eliminate) the need for many connections.
## 2.5 Common Pitfalls
- **Pool too small** → requests queue up, latency spikes under load.
- **Pool too big** → DB melts (Postgres advice: pool size ≈ `(2 × cores) + spindle` ).
- **Forgetting to return** (try-with-resources / `defer`  saves you).
- **Not validating** → dead connections cause first-request-after-network-blip failures.
- **No timeouts on borrow** → callers wait forever.
---

# Part 3: Thread Pooling
## 3.1 What It Is
A **fixed (or bounded) set of worker threads** that pick tasks from a queue, instead of creating a new thread per task. Same idea as connection pooling, applied to threads.

```
Tasks arrive → queue → workers pick up → execute → return to pool
```
## 3.2 Why
- Threads are **expensive**: ~1 MB stack each, kernel scheduling, context switches.
- Creating a thread per request **doesn't scale past a few thousand**.
- Bounded pools provide **natural backpressure** — queue full = caller waits or fails.
## 3.3 Sizing Rules of Thumb
- **CPU-bound work**: pool size ≈ `# cores`  (or `cores + 1` ).
- **I/O-bound work** (blocking on DB/HTTP): much larger; can be hundreds. Formula: (Brian Goetz, _Java Concurrency in Practice_).optimal = cores × target_utilization × (1 + wait_time / compute_time)
- **Mixed**: separate pools per workload (a form of bulkheading).
## 3.4 Queue Choice Matters
- **Bounded queue** (recommended) — pushes back when full.
- **Unbounded queue** (default in many languages) — looks safe, eats all memory under load → OOM.
- **Synchronous handoff** (e.g., `SynchronousQueue` ) — direct handoff, ideal for short-lived bursts.
## 3.5 Bulkheading
Use **separate pools per dependency** (one for DB, one for HTTP to service A, one to service B). A slow downstream can't exhaust threads needed for a healthy one. Covered in resilience patterns.

## 3.6 Modern Twist — Virtual Threads
- **Java 21** introduced virtual threads (Project Loom). Millions of cheap threads scheduled by the JVM onto a small pool of OS threads.
- **Go goroutines, Kotlin coroutines, Erlang processes** — same idea.
- Effect: you can write **straightforward blocking code** that scales like async code. Reduces (but doesn't eliminate) the need for thread pool tuning.
---

# Part 4: Non-blocking I/O
## 4.1 The Old Way (Blocking I/O)
A thread calls `read()` and **sleeps until data arrives**. To handle 10,000 concurrent connections, you need 10,000 threads. Memory + scheduling cost is enormous.

## 4.2 The New Way (Non-blocking I/O)
A thread asks the OS, "tell me when **any** of these sockets is ready." It can manage thousands of connections with **a single thread**.

Built on OS primitives:

- **Linux**: `epoll` 
- **BSD/macOS**: `kqueue` 
- **Windows**: IOCP
A small pool of **event-loop threads** rotates through ready sockets, doing tiny chunks of work for each. Used by:

- **Netty, Vert.x, Akka** (Java)
- **Node.js** (single event loop)
- **Nginx**
- **Go runtime** (under the hood for goroutines)
- **Tokio** (Rust)
## 4.3 Why It Matters
- **C10K and C10M problems** — handle massive connection counts (chat servers, WebSockets, streaming, proxies).
- **Lower memory footprint** per connection.
- **Better CPU cache behavior** — fewer threads, fewer context switches.
## 4.4 The Catch
**Never block the event loop**. A single `Thread.sleep`, slow JDBC call, or CPU-intensive operation freezes **all** connections on that loop. Three rules:

1. Use **non-blocking drivers** (R2DBC instead of JDBC, async HTTP clients).
2. Push CPU-heavy work onto a separate **worker pool**.
3. Be paranoid about accidentally blocking (synchronized blocks, file I/O, logging frameworks doing I/O).
## 4.5 Sync vs Async — Quick Comparison
|  | **Blocking + thread-per-request** | **Non-blocking event loop** |
| ----- | ----- | ----- |
| Code style | Simple, sequential | Callbacks / futures / reactive |
| Memory per connection | High (~1 MB) | Tiny (~KB) |
| Max concurrency | Thousands | Hundreds of thousands |
| Best for | Short, CPU-light requests | Many idle connections (websockets, proxies) |
| Pitfall | Thread exhaustion | Blocking the loop |
Modern runtimes (virtual threads, goroutines) blur this line — you write blocking-style code that behaves non-blockingly underneath.

---

# Part 5: Reactive Programming
## 5.1 What It Is
A programming model built around **asynchronous data streams** with **declarative composition** and **built-in backpressure**. You describe _what to do with data as it arrives_ using operators like `map`, `filter`, `flatMap`, `merge`, `buffer`, `retry`.

## 5.2 The Reactive Streams Spec
Standardized 4 interfaces for interop across libraries:

- **Publisher** — emits items.
- **Subscriber** — receives items.
- **Subscription** — link between them; subscriber **requests N items at a time** (this is backpressure built into the protocol).
- **Processor** — both subscriber and publisher.
Adopted into the JDK as `java.util.concurrent.Flow`.

## 5.3 Implementations
- **Project Reactor** (Spring WebFlux, R2DBC) — `Mono<T>`  and `Flux<T>` .
- **RxJava** — `Observable` , `Single` , `Flowable` .
- **Akka Streams** (Scala/Java).
- **RxJS** (frontend, Angular).
- **Mutiny** (Quarkus).
## 5.4 Example (Reactor)
```java
public Flux<Product> topSellers(int limit) {
    return productRepo.findAll()                    // Flux<Product>
        .filter(p -> p.getStock() > 0)
        .sort(Comparator.comparingInt(Product::getSold).reversed())
        .take(limit)
        .flatMap(this::enrichWithReviews)           // async per-item
        .timeout(Duration.ofSeconds(2))
        .onErrorResume(e -> Flux.empty());
}
```
This pipeline:

- Doesn't block any thread.
- Honors backpressure (downstream signals how much it can consume).
- Composes async error handling, retries, timeouts.
## 5.5 Pros
- Massive concurrency on few threads.
- Composable, declarative.
- Backpressure-aware out of the box.
## 5.6 Cons
- **Steep learning curve** — debugging reactive stacks is genuinely hard.
- Stack traces are useless (operator chains hide call sites).
- Easy to introduce subtle bugs (blocking calls inside operators, hot vs cold publishers, threading model surprises).
- With the rise of **virtual threads** and **coroutines**, plain blocking code now scales similarly without the cognitive overhead — many teams are stepping back from reactive for general business logic.
### When to use reactive
- Stream processing, gateways, proxies, websocket fan-out.
- Chained async I/O with composable error handling.
- High-concurrency, low-CPU services.
### When not to
- Simple CRUD.
- Teams without strong reactive expertise.
- When virtual threads / coroutines give you the same scalability with simpler code.
---

# Part 6: Backpressure _(recap)_
Already covered in detail in the inter-service communication section. Brief recap:

>  **Backpressure** is the mechanism by which a slow consumer signals "slow down" to a fast producer, preventing memory blowup and cascading failure. 

Implementations:

- **Reactive Streams **`**request(n)**`  — pull-style protocol.
- **TCP / HTTP/2 / gRPC flow control** — built-in stream windows.
- **Bounded buffers and queues** — block, drop, or shed when full.
- **Pull-based consumers** (Kafka) — naturally backpressured.
- **Rate limiting / load shedding** at the edge.
- **Adaptive concurrency** (Netflix concurrency-limits).
**Golden rule:** every fast→slow boundary needs an explicit backpressure mechanism. Without one, the slowest part of the system dictates how everything else fails.

---

# Part 7: Async Processing
## 7.1 What It Is
**Decouple the request from the work** — accept the request, return immediately, do the actual work later (in a background thread, queue consumer, or scheduled job).

```
Sync:
     Client → POST /report → [generate 30s] → 200 OK + result
Async:
     Client → POST /report → 202 Accepted + jobId
     ... later ...
     Client → GET /report/jobId → 200 OK + result
                            (or notification / webhook / polling)
```
## 7.2 Patterns
### Fire-and-forget background tasks
For non-critical work: send a welcome email, update analytics. Use a thread pool with a bounded queue.

### Job queue / worker pattern
Push tasks to Redis, RabbitMQ, SQS, Kafka, or a DB-backed queue. Workers consume, process, ack. Tools: **Sidekiq, Celery, BullMQ, Hangfire, Spring Batch, Temporal**.

### Request → polling / webhook callback
Long-running operations (PDF generation, video transcoding). Return a job ID; client polls or receives a webhook on completion.

### Outbox + event-driven
Persist work as events; let downstream services pick them up. Most reliable pattern for cross-service async work.

## 7.3 Why Go Async
- **Faster perceived response** for the user.
- **Decoupling** — caller doesn't fail if downstream is down.
- **Better resource utilization** — long jobs don't tie up HTTP threads.
- **Throughput** — workers scale based on backlog, not request rate.
- **Retries and reliability** — broker handles redelivery.
## 7.4 Watch-outs
- **Eventual consistency** — clients must accept that work isn't done at HTTP return.
- **Lost updates** if not durable (always persist before returning 202).
- **Idempotency required** — at-least-once delivery means duplicates.
- **Observability harder** — need correlation IDs and tracing across hop.
- **Failures are silent** unless you wire up alerts on DLQs / failure metrics.
---

# Part 8: Batching
## 8.1 What It Is
**Group many small operations into one larger one** to amortize fixed costs (network round trip, disk seek, syscall, lock acquisition).

## 8.2 Where It Matters
### Database
```sql
-- 1000 inserts: ~1000 round trips
INSERT INTO users (...) VALUES (...);

-- one batch insert: 1 round trip
INSERT INTO users (...) VALUES (...), (...), (...), ...;
```
Or JDBC `addBatch()` / `executeBatch()`. Often **10–100×** faster.

### Kafka producer
Producers buffer records and flush in batches when `batch.size` or `linger.ms` is hit. Tuning these is the single biggest throughput lever.

### Network / RPC
- HTTP/2 streams + GraphQL aliasing or batching endpoints.
- gRPC bidi streaming.
- Redis `MGET` , `MSET` , `MULTI/EXEC`  pipelining (huge wins — one round trip vs N).
### Logging / metrics
Buffer log lines and flush periodically; same for Prometheus / OTLP exports.

### DataLoader (GraphQL N+1 fix)
Coalesce all `user(id)` requests in one tick into one `SELECT * FROM users WHERE id IN (...)`.

## 8.3 The Trade-off
**Latency vs throughput**. Batching adds wait time (`linger.ms`, buffer fill time) before sending — increases latency for the first request in the batch but vastly increases throughput. Tune to your SLA.

## 8.4 Adaptive Batching
Some systems dynamically size batches based on load — small batches under low load (low latency), big batches under high load (high throughput). Used in Kafka, Spark, and many database engines.

---

# Part 9: Compression
## 9.1 What It Is
Trade **CPU cycles for fewer bytes on the wire or disk** — usually a great deal because networks are slower and more expensive than CPU.

## 9.2 Where to Compress
### HTTP responses
- `Accept-Encoding: gzip, br, zstd`  → server compresses.
- Saves 60–90% on JSON, HTML, CSS, JS.
- Modern stack: **Brotli (br)** for text → smaller than gzip; **zstd** for binary.
### Inter-service / gRPC
- gRPC supports per-message compression (gzip, identity, custom).
- Big wins for payloads >1 KB.
### Kafka
- Producer-side compression: `snappy` , `lz4` , `zstd` , `gzip` .
- Compresses entire batches end-to-end (broker stores compressed; consumer decompresses).
- Often **2–5×** higher throughput, lower disk + network usage.
### Storage
- Columnar formats (**Parquet, ORC**) compress per column → exceptional ratios for analytics.
- Compressed page storage in MySQL InnoDB, Postgres TOAST, MongoDB WiredTiger.
## 9.3 Choosing an Algorithm
| Algorithm | Speed | Ratio | Best for |
| ----- | ----- | ----- | ----- |
| **gzip** | Medium | Good | Universal default |
| **snappy** | Very fast | Modest | Kafka, RPC, low-latency |
| **lz4** | Very fast | Modest | Kafka, in-memory |
| **zstd** | Fast | Excellent | Modern default; tunable levels |
| **brotli** | Slow encode, fast decode | Best for text | HTTP responses |
**zstd** has become the modern default — comparable speed to lz4, ratio close to gzip, often better than both.

## 9.4 Watch-outs
- **CPU cost** — measure; not free.
- **Don't compress already-compressed data** (images, videos, encrypted content) — wastes CPU, may even grow.
- **Small payloads** (<~1 KB) often not worth it.
- **HTTPS + compression of secrets** = **CRIME/BREACH** attacks; mitigated by modern TLS, but worth knowing.
---

# Part 10: Pagination
## 10.1 The Problem
Returning 1 million rows in a single response:

- Crushes the server (memory).
- Crushes the client (parsing, rendering).
- Crushes the network.
- Times out.
## 10.2 Two Main Strategies
### Offset-based pagination
```sql
SELECT * FROM orders ORDER BY created_at LIMIT 20 OFFSET 1000;
```
- ✅ Simple; supports arbitrary page jumps ("page 47").
- ❌ **Gets slower as offset grows** — DB must scan and discard offset rows. `OFFSET 1000000`  is brutal.
- ❌ **Inconsistent under writes** — inserts/deletes shift items, causing duplicates or skips between pages.
- ❌ Doesn't shard well.
### Cursor-based (keyset) pagination
```sql
SELECT * FROM orders
WHERE (created_at, id) < (?, ?)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```
The client passes the **last seen value as a cursor** (often opaque/encoded).

- ✅ **Constant time** regardless of page depth.
- ✅ **Stable under inserts/deletes**.
- ✅ Works on sharded data.
- ❌ Can't jump to "page 47" — only forward/backward.
- ❌ Requires a stable, indexed sort key (often `(timestamp, id)`  for uniqueness).
## 10.3 Relay-Style Connections (the GraphQL standard)
```graphql
posts(first: 20, after: "cursor123") {
  edges { cursor, node { id, title } }
  pageInfo { hasNextPage, endCursor }
}
```
Combines cursor-based pagination with metadata for infinite scroll.

## 10.4 General Rules
- **Always cap page size** — even if user asks for 1000, max it at 100.
- **Default to cursor-based** for any list that can grow unbounded.
- **Index your sort key** — without it, both methods are slow.
- For **infinite scroll**, cursor pagination is mandatory.
---

# Part 11: Streaming APIs
## 11.1 What It Is
Send (or receive) data as a **continuous flow** rather than as one big response. The server starts producing output before it's all ready; the client starts consuming before the response is complete.

```
Bulk:       [-------- 50 MB response ---------]
                                          ^ client waits for everything
Streaming:  [chunk][chunk][chunk][chunk]...
                ^ client processes as they arrive
```
## 11.2 When to Use
- **Large result sets** — exporting millions of rows; don't materialize in memory.
- **Real-time feeds** — chat, notifications, stock tickers, live logs.
- **AI/LLM responses** — token-by-token streaming improves perceived latency.
- **File uploads/downloads** — chunked transfer, resumable.
- **Continuous data** — telemetry, analytics events.
## 11.3 Protocols and Tools
| Protocol | Direction | Best for |
| ----- | ----- | ----- |
| **HTTP chunked transfer encoding** | Server → Client | Large responses, log streams |
| **Server-Sent Events (SSE)** | Server → Client | Notifications, live feeds, LLM tokens |
| **WebSockets** | Bi-directional | Chat, multiplayer, collaboration |
| **gRPC streaming** | Server, Client, or Bi-di | Service-to-service |
| **Kafka / Pulsar** | Producer → Consumer | Inter-service event streams |
| **HTTP/2 push & HTTP/3** | Server → Client | Modern alternatives |
## 11.4 Server-Side Implementation Tips
- **Don't load the whole result set into memory** — use cursors, iterators, or `Flux<T>`  / `IAsyncEnumerable<T>` .
- **Flush periodically** (per chunk, not per byte).
- **Heartbeats** for SSE/WebSockets to detect dead connections.
- **Backpressure** — if the client is slow, you must slow down (TCP flow control gives you this for free, mostly).
- **Idempotency / resumability** — clients reconnect; let them resume from a cursor / last event ID.
## 11.5 Client-Side
- Process records as they arrive, don't wait for end-of-response.
- Handle reconnects gracefully.
- Honor `Last-Event-ID`  (SSE) or offsets (Kafka) on resume.
## 11.6 Streaming vs Batching — Different Optimizations
- **Batching** = group many small things into one big trip → fewer round trips, higher throughput.
- **Streaming** = send one big thing as many small chunks → lower memory, faster time-to-first-byte.
They complement each other.

---

# Part 12: How These Patterns Stack
A high-throughput service typically uses **most of these at once**:

```
Client
  │  (HTTP/2, gzip/brotli compression, cursor pagination)
  ▼
API Gateway
  │  (connection pool, rate limit, batching of small calls)
  ▼
App Server
  │  Reactive / async pipelines, backpressure-aware
  │  Bounded thread pools (or virtual threads)
  │  Non-blocking IO event loop (Netty / Tokio)
  │
  ├──► DB (HikariCP pool, batch INSERTs, statement cache)
  ├──► Cache (Redis pipeline, near cache for hot keys)
  ├──► Kafka (batched, lz4/zstd compressed, async produce)
  └──► Background workers (async processing, idempotent consumers)

Long responses → Streaming APIs (SSE, gRPC streaming) instead of bulk.
```
Every tier:

- **Pools** something (connections, threads).
- **Batches** wherever feasible.
- **Compresses** payloads when they're big.
- **Paginates / streams** instead of dumping everything at once.
- **Honors backpressure** end-to-end.
---

# Part 13: When To Optimize (and When Not To)
>  "Premature optimization is the root of all evil." — Knuth 

Stick to this discipline:

1. **Measure first** — profile (CPU, allocations, I/O), benchmark, track p99 latency. Without numbers, you'll optimize the wrong thing.
2. **Optimize the bottleneck** — if a service spends 90% of its time waiting on DB, faster JSON parsing won't help.
3. **Sequence of leverage**:
    - **Architecture / algorithm** changes (10–1000×).
    - **I/O optimization** — pooling, batching, async (2–100×).
    - **Caching** (10–1000× for hot reads).
    - **Compression / payload size** (2–10×).
    - **Code-level micro-optimization** (rarely needed; <2×).

4. **Budget your latency** — distribute SLA across hops; know where you spend it.
5. **Use the cheapest pattern that meets the SLO** — virtual threads beat reactive complexity for most teams now; cursor pagination beats offset; gzip beats no compression.
---

# TL;DR
- **Connection pooling** — never create per request what you can reuse; protects DBs from connection storms.
- **Thread pooling** — bounded pools with bounded queues; size by workload type. Virtual threads / goroutines now reduce manual tuning.
- **Reactive programming** — composable async streams with built-in backpressure; powerful but complex; weigh against simpler alternatives.
- **Non-blocking I/O** — single threads handle thousands of connections via `epoll` /`kqueue` ; never block the event loop.
- **Backpressure** — make slow consumers throttle fast producers; without it, the whole system collapses on the slowest part.
- **Async processing** — decouple request from work; return fast, do later; needs idempotency and observability.
- **Batching** — group small operations into big ones to amortize fixed costs (DB, Kafka, RPC, Redis pipelines, GraphQL DataLoader).
- **Compression** — trade CPU for bytes; modern default is **zstd** (or **brotli** for HTTP text); skip already-compressed data.
- **Pagination** — always cap, prefer **cursor-based** over offset for unbounded lists; index your sort keys.
- **Streaming APIs** — for large/continuous data, send as chunks (SSE, WebSockets, gRPC streams, HTTP chunked) instead of materializing everything; pair with backpressure and heartbeats.
>  Performance comes from a **stack of small, deliberate decisions** — pool, batch, compress, stream, paginate, async, backpressure — applied at every layer. None alone is magic; together they're the difference between a system that survives production and one that doesn't. 





<!--- Eraser file: https://app.eraser.io/workspace/bciRDxY6PbaqC6harEo4 --->