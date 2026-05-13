<p><a target="_blank" href="https://app.eraser.io/workspace/1RX04zRINCpqwIOGXAls" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>

A practical reference for the patterns that actually appear in production microservice systems, why each exists, and when to use it. Some of these patterns have already been covered in detail in earlier discussions; for those, I'll give a focused recap and emphasize how they fit into the broader pattern catalog.

---

# Part 1: How These Patterns Fit Together
Microservices patterns roughly fall into five categories:

| Category | Patterns |
| ----- | ----- |
| **Data & consistency** | Saga, CQRS, Event Sourcing, Outbox, Database per Service |
| **Communication** | API Gateway, BFF, Aggregator, Anti-Corruption Layer |
| **Resilience** | Circuit Breaker, Bulkhead, Retry |
| **Deployment / runtime** | Sidecar, Ambassador, Adapter |
| **Migration / evolution** | Strangler Fig, Anti-Corruption Layer |
A real system uses **many of them at once**. They are complementary, not alternatives.

---

# Part 2: Data & Consistency Patterns
## 2.1 Saga Pattern _(recap)_
A **sequence of local transactions** across multiple services. Each step publishes an event/command that triggers the next; failures trigger **compensating transactions** to undo previous steps. There is no global lock, no global commit.

Two flavors:

- **Choreography** — services react to events; no central coordinator. Loosely coupled, but workflow is implicit and hard to trace.
- **Orchestration** — a central orchestrator (Temporal, Camunda, Step Functions) drives the workflow as a state machine. Explicit, observable, easier to evolve.
**Use it whenever** a single business operation spans multiple services and you need eventual consistency without 2PC.

## 2.2 CQRS — Command Query Responsibility Segregation
### What it is
Separate the **write model** (commands) from the **read model** (queries). They have different shapes, different storage, and often different consistency requirements.

```
┌─────────────┐
Command  ───►  │ Write Model │  ───► writes to write store (normalized)
               └──────┬──────┘
                      │ emits events
                      ▼
               ┌─────────────┐
               │ Projector   │
               └──────┬──────┘
                      │ updates read store(s)
                      ▼
               ┌─────────────┐
Query    ◄───  │ Read Model  │  (denormalized, optimized for queries)
               └─────────────┘
```
### Why
- **Different optimization needs** — writes need consistency and validation; reads need speed and flexibility.
- **Independent scaling** — read side often dwarfs writes; scale them separately.
- **Multiple read models** for different consumers (search index, analytics view, mobile API).
- **Pairs naturally with Event Sourcing** — events from the write side feed projections.
### Trade-offs
- **Eventual consistency** between command and query sides.
- More moving parts than a CRUD service.
- Schema duplication; projections must be rebuildable.
### When to use
- Read/write workloads have **very different scale or shape**.
- Need multiple specialized read views.
- Already using event sourcing or event-driven communication.
### When not to use
- Simple CRUD; CQRS is overkill.
## 2.3 Event Sourcing _(recap)_
Store **events (state changes) as the source of truth** instead of current state. Current state = `fold(events)`.

- Naturally pairs with Kafka (durable, ordered, replayable log).
- Enables full audit, time travel, replay, multiple projections.
- Best for banking, orders, inventory, compliance, anywhere history matters.
- **Avoid for simple CRUD** — overhead is real.
## 2.4 Outbox Pattern _(recap)_
Solves the **dual-write problem**: writing to your DB _and_ publishing an event must happen atomically, but they're two systems.

```sql
BEGIN;
  INSERT INTO orders (...);
  INSERT INTO outbox (event_id, payload);
COMMIT;
```
A relay (CDC tool like **Debezium**, or a polling worker) publishes the outbox rows to Kafka. Combined with idempotent consumers (Inbox), gives **effectively-once** end-to-end delivery without distributed transactions.

**Use whenever a service must publish events reliably alongside DB changes — which is almost every event-driven service.**

## 2.5 Database per Service
### What it is
**Each microservice owns its own database**, and **no other service may access it directly**. Other services must go through the owning service's API or consume its events.

```
┌────────────┐    ┌────────────┐    ┌────────────┐
│  Orders    │    │ Payments   │    │ Inventory  │
│  Service   │    │  Service   │    │  Service   │
└─────┬──────┘    └─────┬──────┘    └─────┬──────┘
      │                 │                 │
   ┌──▼──┐           ┌──▼──┐           ┌──▼──┐
   │ DB  │           │ DB  │           │ DB  │
   │ (PG)│           │ (PG)│           │(Mongo)│
   └─────┘           └─────┘           └─────┘
```
### Why
- **Loose coupling** — services don't share schemas; they can evolve independently.
- **Right tool for the job** — each service can use the DB type that fits (relational, document, graph, key-value, time-series).
- **Independent scaling and failure isolation** — one service's DB outage doesn't take down others.
- **Clear ownership** — a single team owns the schema and data.
### Trade-offs
- **No cross-service joins** — must use API composition or read models (CQRS).
- **Distributed transactions** are out — use sagas + outbox.
- **Data duplication** — same entity may appear in multiple services in different forms.
- **Operational overhead** — many DBs to back up, monitor, upgrade.
### Common anti-patterns to avoid
- "Shared database" between two services — couples them at the worst possible layer.
- Allowing one service to read another service's tables directly — same problem.
- Synchronous chains of API calls to "join" data — slow and fragile.
**Database per service is foundational** — without it, you don't really have microservices, you have a distributed monolith.

---

# Part 3: Communication Patterns
## 3.1 API Gateway Pattern
### What it is
A **single entry point** in front of all microservices that handles cross-cutting concerns — authentication, TLS, rate limiting, routing, request transformation, caching, observability.

```
Internet
           │
           ▼
┌──────────────────────┐
│   API Gateway        │  ← auth, rate limit, routing, TLS, logs
└──┬─────┬─────┬─────┬─┘
   ▼     ▼     ▼     ▼
orders payments users notifs   (microservices)
```
### Why
- Clients see **one URL and one auth flow**, not one per service.
- Cross-cutting concerns are centralized.
- Backend services can stay private and untrusted-by-default.
- Easier protocol translation (gRPC inside, REST/GraphQL outside).
### Examples
- **Kong, NGINX, Envoy, Traefik, AWS API Gateway, Apigee, Spring Cloud Gateway.**
### Trade-offs
- **Single point of failure** — must be HA, scalable, well-monitored.
- Risk of becoming a "god service" with too much logic.
- Adds a network hop.
### When to use
- Almost always for any non-trivial microservice system.
## 3.2 Backend for Frontend (BFF)
### What it is
A **dedicated backend per client type** (web, iOS, Android, internal tools), each one tailored to that client's needs. The BFF aggregates calls to multiple downstream microservices and returns data in the exact shape the frontend needs.

```
Web app    ──►  Web BFF
iOS app    ──►  iOS BFF       ──►  microservices (orders, users, …)
Android    ──►  Android BFF
Partner    ──►  Partner BFF
```
### Why
- Different clients have **different data needs, payload sizes, latency budgets** — one generic API can't serve them all well.
- Mobile cares about small payloads, batched calls, image variants; web cares about SEO, server rendering.
- Frontend teams can **own their BFF** and ship without coordination with backend teams.
### Trade-offs
- **More services to maintain.**
- Risk of duplicating logic across BFFs (extract shared libs / call common services).
- Don't put domain logic here — BFFs are aggregation/transformation layers, not business logic owners.
### When to use
- Multiple very different clients consuming the same backend.
- A generic API forces clients to over-fetch or make many roundtrips.
- Often replaced by **GraphQL gateway** for similar reasons.
## 3.3 Aggregator Pattern
### What it is
A service whose job is to **call multiple downstream services and combine their responses** into a single response for the caller.

```
┌──────────────┐
Client ───►    │  Aggregator  │
               └──┬────┬────┬─┘
                  ▼    ▼    ▼
               user  order  shipping
               svc   svc    svc
```
This is the read-time equivalent of "API Composition" (vs CQRS read models).

### Why
- Reduces client roundtrips.
- Hides complexity of multiple backend services.
- Useful for read-heavy "details" pages that need data from many sources.
### Trade-offs
- **As slow as the slowest dependency** (unless calls are parallel).
- All dependencies must be available — fragile.
- Doesn't scale for very complex queries — switch to **CQRS read models** or GraphQL.
### Often appears as
- A BFF.
- A specialized "search/details" microservice.
- An orchestrator within an API gateway plugin.
## 3.4 Anti-Corruption Layer (ACL)
### What it is
A **translation layer between two bounded contexts** that prevents the model of one system from polluting another. The ACL converts external models into your domain's models (and vice versa), shielding you from upstream changes.

```
Your service ───► ACL ───► External / legacy system
       │
translates models, names,
terminology, error codes
```
### Why
- **Legacy systems** often have ugly, sprawling, inconsistent models you don't want to inherit.
- **Third-party APIs** change; the ACL absorbs the change in one place.
- **Multiple bounded contexts** (DDD) need clear translations to remain decoupled.
- Lets you **rewrite gradually** (works hand-in-hand with the Strangler Fig pattern).
### Examples
- A service consuming a legacy mainframe SOAP API but exposing a clean REST/event interface internally.
- Wrapping Salesforce / SAP behind a domain-specific façade.
### Trade-offs
- More code to maintain.
- Adds latency.
- Can become a bottleneck if the ACL itself becomes a god service.
### When to use
- Integrating with legacy systems, third-party APIs, or another team's bounded context that doesn't match yours.
---

# Part 4: Resilience Patterns
## 4.1 Circuit Breaker
### What it is
Automatically stop calling a failing downstream service after a threshold of failures, **fail fast** for a cooldown period, then probe whether it has recovered. Prevents cascading failures.

States:

- **Closed** — normal; requests flow.
- **Open** — failure threshold exceeded; calls fail immediately for a cooldown period.
- **Half-open** — test a few calls; if they succeed, close the breaker; if they fail, re-open.
```
failures > threshold
Closed ──────────────────► Open
  ▲                         │
  │ success                 │ cooldown elapsed
  │                         ▼
  └────────────────── Half-Open
                           │
                    failure │
                           ▼
                           Open
```
### Why
A slow or dead downstream is the #1 cause of cascading microservice outages — threads pile up waiting on doomed calls and **everything** falls over. Circuit breakers convert slow failures into fast failures, freeing resources.

### Tools
- **Resilience4j** (Java), **Polly** (.NET), **gobreaker** (Go), **opossum** (Node).
- Built into service meshes (Envoy outlier detection).
### Watch-outs
- Set thresholds based on real traffic patterns; too sensitive = flapping.
- Combine with **fallbacks** (cached value, default, degraded experience).
- Always pair with **timeouts**; a circuit breaker without timeouts is useless.
## 4.2 Bulkhead
### What it is
**Isolate resources** so that one failing dependency cannot exhaust resources used by other dependencies. Named after watertight compartments in ships — one flooded bulkhead doesn't sink the boat.

### Implementations
- **Separate thread pools / connection pools** per downstream.
- **Semaphores** capping concurrent calls per dependency.
- **Separate processes / containers** per workload.
- **Separate Kubernetes namespaces / node pools** for noisy neighbors.
### Example without bulkheading
A service with one shared 200-thread pool calls Service A (fast) and Service B (suddenly slow). All 200 threads end up blocked on B → calls to A also fail because no threads are left.

### With bulkheading
A gets 100 threads, B gets 100 threads. B's slowdown only kills calls to B; A is unaffected.

### Tools
- Resilience4j Bulkhead, Hystrix (deprecated), Netflix concurrency-limits, service mesh connection pool limits.
## 4.3 Retry
### What it is
Automatically retry failed requests, usually with **exponential backoff and jitter** and a cap on attempts.

### Critical rules
- **Only retry idempotent operations** — `GET` , `PUT` , `DELETE` , or anything with an idempotency key. Retrying a non-idempotent `POST`  may double-charge a customer.
- **Use exponential backoff with jitter** to avoid synchronized retry storms.
- **Cap total time** — retries shouldn't compound forever.
- **Combine with circuit breaker** — when the breaker is open, don't retry at all.
- **Retry only at one layer** — if A retries B which retries C, a failure in C produces 27× normal load.
### Example (Resilience4j)
```java
RetryConfig config = RetryConfig.custom()
.maxAttempts(3)
.intervalFunction(IntervalFunction.ofExponentialRandomBackoff(200, 2.0))
.retryOnException(e -> e instanceof IOException)
.build();
```
### Mesh-side
Service meshes (Istio, Linkerd) provide retries declaratively — same rules apply.

---

# Part 5: Runtime / Deployment Patterns
These are deployment patterns where helper containers run alongside the main app, typically inside the same Kubernetes pod. Already covered in detail in the deployment & infrastructure section — quick recap:

## 5.1 Sidecar Pattern _(recap)_
A helper container in the **same pod** as the application — log shipper, metrics exporter, secrets agent, service-mesh proxy. Decouples cross-cutting concerns from app code; works for any language.

## 5.2 Ambassador Pattern _(recap)_
A specialized sidecar that **proxies outbound traffic** from the app to an external service (DB, third-party API) — handling TLS, retries, service discovery, connection pooling. The app simply talks to `localhost`.

## 5.3 Adapter Pattern _(recap)_
A sidecar that **normalizes what the app exposes** to the platform — e.g., translating app-specific metrics into Prometheus format, or app logs into structured JSON. Standardizes legacy or third-party apps without modifying them.

(Quick distinction: **Sidecar** = generic helper. **Ambassador** = outbound proxy. **Adapter** = inbound interface translator.)

---

# Part 6: Evolution & Migration Patterns
## 6.1 Strangler Fig Pattern
### What it is
A pattern for **incrementally migrating a monolith to microservices** (or any legacy system to a new one) without a big-bang rewrite. Named after the strangler fig vine that grows around a tree, eventually replacing it.

You put a **routing layer** (proxy / gateway) in front of the legacy system. New functionality and migrated functionality go to new services; everything else keeps going to the legacy system. Over time, more routes shift to the new system, until the legacy system is gone.

```
Step 0:    Client → Monolith
Step 1:    Client → Proxy ──► Monolith   (everything still goes there)
                            └──► (nothing yet)
Step 2:    Client → Proxy ──► Monolith   (most routes)
                            └──► New service for /orders/*
Step 3:    Client → Proxy ──► Monolith   (a few routes)
                            └──► New services for /orders, /users, /payments
Step N:    Client → Proxy ──► (nothing)
                            └──► All new services
```
### Why
- **No big-bang rewrite** — those almost always fail.
- **Continuous delivery of value** — each migrated piece is shipped independently.
- **Reduced risk** — easy rollback per slice.
- **Old and new run side-by-side** during migration.
### Supporting techniques
- **Anti-Corruption Layer** between new services and the old monolith.
- **Outbox pattern** to publish events from the monolith DB to new services (often via Debezium CDC).
- **Feature flags** to switch users between old and new gradually.
- **Shadow traffic / dual-writes** to validate new behavior before cutover.
### Trade-offs
- **Long-lived migrations** — months or years.
- **Two systems to operate** during transition.
- **Discipline required** — easy to leave the migration "90% done" forever.
### When to use
- **Almost any non-trivial monolith → microservices migration.** It's the de-facto standard approach.
## 6.2 Anti-Corruption Layer _(see Part 3.4)_
Often used hand-in-hand with the Strangler Fig — the ACL is what prevents the legacy system's model from infecting your new services during the migration.

---

# Part 7: How They All Fit in a Real System
A typical e-commerce checkout combines many of these patterns at once:

```
Mobile client ─► Mobile BFF ─┐
Web client    ─► Web BFF    ─┤
                              ▼
                       API Gateway
                       (auth, rate limit, TLS)
                              │
           ┌──────────────────┼────────────────────┐
           ▼                  ▼                    ▼
      Orders Svc          Payments Svc        Inventory Svc
     (DB-per-svc)        (DB-per-svc)         (DB-per-svc)
           │                  │                    │
           │  publishes via   │  publishes via     │
           │  Outbox →        │  Outbox →          │
           ▼                  ▼                    ▼
                         Kafka topics
                              │
               ┌──────────────┼─────────────────┐
               ▼              ▼                 ▼
         Saga Orchestrator   CQRS read models   Anti-Corruption
         (Temporal)          (search, analytics) Layer → legacy ERP

Every service:
  • Sidecar proxy (service mesh): mTLS, retries, observability
  • Bulkheads + circuit breakers around external calls
  • Idempotent consumers (Inbox) for at-least-once Kafka delivery
  • Strangler Fig in progress: legacy monolith still serves /admin/*
```
---

# Part 8: Pattern Selection Cheat Sheet
| If you need to… | Use |
| ----- | ----- |
| Coordinate a multi-service workflow | <p>**Saga**</p><p> (choreography or orchestration)</p> |
| Separate read/write models for scale | **CQRS** |
| Audit, time-travel, or replay state | **Event Sourcing** |
| Reliably publish events with DB writes | **Outbox** |
| Isolate data ownership per service | **Database per Service** |
| Provide a single front door for clients | **API Gateway** |
| Tailor backend per client type | **BFF** |
| Combine data from multiple services | <p>**Aggregator**</p><p> (or CQRS read model)</p> |
| Integrate with legacy/third-party | **Anti-Corruption Layer** |
| Stop cascading failures | **Circuit Breaker** |
| Isolate resource pools per dependency | **Bulkhead** |
| Tolerate transient failures | **Retry (with backoff + jitter)** |
| Add platform features without app changes | <p>**Sidecar**</p><p> (and </p><p>**Ambassador / Adapter**</p><p>)</p> |
| Migrate a monolith incrementally | <p>**Strangler Fig**</p><p> + </p><p>**ACL**</p> |
---

# Part 9: Universal Principles
Regardless of which patterns you choose, these principles run through all of them:

1. **Design for failure** — every network call will eventually fail, time out, or duplicate.
2. **Idempotency is a virtue** — make consumers and APIs idempotent; it makes everything else simpler.
3. **Eventual consistency is the default** — strong consistency across services is expensive and often unnecessary.
4. **Loose coupling beats clever coordination** — events and clear contracts beat shared databases and synchronous chains.
5. **Observability isn't optional** — distributed tracing, metrics, structured logs are how you survive in production.
6. **Pattern stacking is normal** — Outbox + Saga + CQRS + Circuit Breaker + Sidecar in one service is unremarkable.
7. **Don't adopt patterns prematurely** — every pattern adds complexity; each must earn its place.
---

# TL;DR
Microservice patterns aren't a buffet to pick from arbitrarily — they're **answers to specific distributed-systems problems**:

- **Saga, CQRS, Event Sourcing, Outbox, Database-per-Service** solve **data and consistency** problems.
- **API Gateway, BFF, Aggregator, Anti-Corruption Layer** solve **communication** problems.
- **Circuit Breaker, Bulkhead, Retry** solve **resilience** problems.
- **Sidecar, Ambassador, Adapter** solve **runtime / cross-cutting concerns** without touching app code.
- **Strangler Fig (+ ACL)** solves the **migration** problem.
Use the ones you actually need, combine them deliberately, and treat patterns as **tools that add complexity** — only adopt each one when the pain it solves outweighs the pain it introduces.



<!--- Eraser file: https://app.eraser.io/workspace/1RX04zRINCpqwIOGXAls --->