<p><a target="_blank" href="https://app.eraser.io/workspace/XtalLcp56TqNdwObjDOz" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>



>  A comprehensive deep-dive into the **core fundamentals of microservices**, with real-world examples (Netflix, Amazon, Uber, Spotify) and the exact mental models, trade-offs, and interview talking points expected at the senior / staff engineer level. 

---

## Table of Contents
1. Monolith vs Microservices
2. Distributed Systems Basics
3. Service Decomposition Strategies
4. Domain-Driven Design (DDD)
5. Bounded Context
6. Single Responsibility Principle
7. Database per Service
8. Shared Database Anti-pattern
9. Stateless Services
10. Service Granularity
11. Loose Coupling
12. High Cohesion
13. API-first Design
14. Backend for Frontend (BFF)
15. Micro Frontends
---

## 1. Monolith vs Microservices
### 1.1 What is a Monolith?
A **monolith** is a single deployable unit containing **all** business functionality — UI, business logic, data access — typically packaged as one WAR/JAR/EXE and running in one process.

```
┌──────────────────────────────────────┐
│         MONOLITHIC APPLICATION       │
│  ┌──────┐  ┌──────┐  ┌──────────┐    │
│  │Orders│  │Users │  │Inventory │    │
│  └──────┘  └──────┘  └──────────┘    │
│  ┌────────────────────────────────┐  │
│  │     Single Shared Database     │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
```
### 1.2 What are Microservices?
Microservices are an **architectural style** where a system is composed of **small, independent, loosely-coupled services**, each owning its data, deployed independently, communicating over the network (HTTP/gRPC/messaging).

```
┌────────┐    ┌────────┐    ┌──────────┐
│ Orders │←→ │ Users  │←→ │Inventory │
│ Svc+DB │    │ Svc+DB │    │  Svc+DB  │
└────────┘    └────────┘    └──────────┘
```
### 1.3 Side-by-Side Comparison
| Aspect | Monolith | Microservices |
| ----- | ----- | ----- |
| **Deployment** | One big artifact | Independent per service |
| **Scaling** | Vertical (whole app) | Horizontal per service |
| **Database** | Shared | Per service |
| **Tech stack** | Uniform | Polyglot allowed |
| **Team structure** | Centralized | Autonomous teams (2-pizza) |
| **Failure isolation** | Bug crashes everything | Bug isolated to one service |
| **Initial complexity** | Low | High (network, DevOps) |
| **Long-term scaling** | Hard | Built for it |
| **Transactions** | ACID, easy | Distributed (Saga, eventual consistency) |
| **Debugging** | Stack trace in one process | Distributed tracing required |
| **Latency** | In-process calls (ns) | Network calls (ms) |
### 1.4 Real-World Examples
- **Netflix (2008–2012)** — Famously migrated from a Oracle-backed monolith to ~700 microservices on AWS after a database corruption caused 3 days of downtime.
- **Amazon (~2002)** — Bezos's "API mandate" forced every team to expose data only via service interfaces.
- **Shopify** — Famously runs a **modular monolith** (Rails) at massive scale; proof that "monolith" isn't always wrong.
- **Etsy** — Long-time monolith advocate; eventually decomposed selectively.
### 1.5 When to Choose Which?
**Start with a monolith if:**

- Small team (< 10 engineers).
- Domain not well understood.
- Need to move fast and validate the product.
- Low operational maturity (no Kubernetes, no observability).
**Move to microservices when:**

- Teams stepping on each other's toes during deploys.
- Different parts of the system need different scaling profiles.
- You can invest in DevOps, observability, and CI/CD.
- Independent release cycles are required.
### 1.6 Senior Talking Point
>  _"Microservices are an organizational pattern as much as a technical one. They optimize for _**_team autonomy and independent deployability_**_ at the cost of operational complexity. If you can't pay that operational tax, a well-modularized monolith is almost always the better choice."_ 

---

## 2. Distributed Systems Basics
Microservices are **distributed systems** — and distributed systems have well-known, brutal trade-offs.

### 2.1 Fallacies of Distributed Computing (Peter Deutsch)
Things naive developers wrongly assume:

1. The network is reliable.
2. Latency is zero.
3. Bandwidth is infinite.
4. The network is secure.
5. Topology doesn't change.
6. There is one administrator.
7. Transport cost is zero.
8. The network is homogeneous.
Every microservices design must respect all 8.

### 2.2 The CAP Theorem
Given a network partition, you can choose **two of three**:

- **C**onsistency — every read returns the latest write.
- **A**vailability — every request gets a response.
- **P**artition tolerance — system continues despite network splits.
In practice you can't trade away P (networks fail), so it's really **CP vs AP**:

| System Type | Examples | Trade-off |
| ----- | ----- | ----- |
| **CP** | MongoDB (default), HBase, Zookeeper, etcd | Refuses reads/writes during partition |
| **AP** | Cassandra, DynamoDB, Riak | Returns possibly-stale data |
### 2.3 PACELC — A More Complete Model
- During **P**artition → choose **A**vailability or **C**onsistency.
- **E**lse (normal operation) → choose **L**atency or **C**onsistency.
This explains DynamoDB (PA/EL) vs Spanner (PC/EC).

### 2.4 Consistency Models
| Model | Meaning |
| ----- | ----- |
| **Strong** | All reads see the last write |
| **Eventual** | Reads will converge, eventually |
| **Causal** | Causally related writes seen in order |
| **Read-your-writes** | A user sees their own changes |
>  In microservices, **eventual consistency is the norm** — embrace it via events, Sagas, idempotency. 

### 2.5 Failure Modes Unique to Distributed Systems
- **Partial failure** — one service is down; how do you respond?
- **Network partition** — half the cluster can't see the other half.
- **Slow service is worse than dead service** — ties up callers.
- **Cascading failures** — one slow dependency saturates the whole system.
- **Clock skew** — no two machines agree on time.
- **Duplicate messages** — at-least-once delivery means retries.
### 2.6 Essential Patterns to Defend Against Failure
| Pattern | Purpose |
| ----- | ----- |
| **Timeouts** | Bound waiting on remote calls |
| **Retries with backoff + jitter** | Avoid retry storms |
| **Circuit breakers** | Stop hammering a failing dependency |
| **Bulkheads** | Isolate resources so one failure doesn't cascade |
| **Idempotency keys** | Make retries safe |
| **Rate limiting / throttling** | Protect downstream |
| **Health checks + auto-restart** | Replace dead instances |
| **Graceful degradation** | Serve a degraded response, not a 500 |
### 2.7 Real-World Example: Netflix
Netflix experiences **billions** of inter-service calls per day. They use:

- **Hystrix** (circuit breakers) — now superseded by **Resilience4j**.
- **Eureka** — service discovery.
- **Ribbon** — client-side load balancing.
- **Chaos Monkey** — intentionally kills services in production to test resilience.
### 2.8 Senior Talking Point
>  _"Anything you call over the network can fail, be slow, return duplicates, or appear to succeed when it didn't. A senior engineer's job is to design with these realities as the default, not the exception."_ 

---

## 3. Service Decomposition Strategies
How do you split a monolith into microservices? There are several proven strategies — usually combined.

### 3.1 Decompose by Business Capability
Group functionality around a business capability the company performs:

- `OrderManagement` 
- `InventoryManagement` 
- `Pricing` 
- `Shipping` 
- `Notification` 
Mirrors the **org structure** (Conway's Law).

### 3.2 Decompose by Subdomain (DDD)
Use Domain-Driven Design to find **bounded contexts** and align services to them. (Covered in detail in §4–5.)

### 3.3 Decompose by Verb/Use-Case
Each service handles a single user-facing action:

- `CheckoutService` 
- `RegisterUserService` 
- `SubmitOrderService` 
⚠️ Often too fine-grained → can become "nano-services."

### 3.4 Strangler Fig Pattern (Martin Fowler)
The safest way to migrate from a monolith:

1. Put a **proxy/API gateway** in front of the monolith.
2. Build a new microservice for one capability.
3. Route those requests to the new service.
4. Slowly "strangle" the monolith feature by feature.
5. When the monolith does nothing useful, delete it.
```
Client → API Gateway →┬→ New Service (orders)
└→ Monolith (everything else)
```
Real-world example: **Amazon, Shopify, and Stripe** have all used variants of this for years.

### 3.5 Decompose by Data Ownership
Identify which entity each piece of code "owns." Each entity = one service.

### 3.6 Anti-patterns
- **Decomposing by technical layer** (`UI Service` , `Logic Service` , `DB Service` ) — this is a distributed monolith.
- **Over-decomposition** — 50 services for 5 engineers.
- **CRUD-per-table services** — meaningless service boundaries.
### 3.7 Senior Talking Point
>  _"Service boundaries should match business boundaries, not technical layers. The single biggest signal you've split badly is when a single feature requires changes across 5 services."_ 

---

## 4. Domain-Driven Design (DDD)
### 4.1 What is DDD?
DDD (Eric Evans, 2003) is a methodology for designing software **around the business domain** using a **ubiquitous language** shared by developers and domain experts.

It's the **gold standard for finding microservices boundaries.**

### 4.2 Core Building Blocks
| Concept | Meaning |
| ----- | ----- |
| **Domain** | The problem space (e.g., e-commerce) |
| **Subdomain** | A coherent part (Orders, Payments, Shipping) |
| **Bounded Context** | A boundary inside which a model is consistent |
| **Ubiquitous Language** | Vocabulary shared between business and devs |
| **Entity** | Object with identity (`Order #123`) |
| **Value Object** | Immutable, identity-less (`Money(USD, 10)`) |
| **Aggregate** | <p>Cluster of entities treated as one unit; has an </p><p>**aggregate root**</p> |
| **Repository** | Abstraction over persistence |
| **Domain Event** | Something significant that happened (`OrderPlaced`) |
### 4.3 Strategic vs Tactical DDD
- **Strategic DDD** — defining bounded contexts, subdomains, context maps. (What microservices to build.)
- **Tactical DDD** — entities, aggregates, value objects. (How to model each service internally.)
For microservices design, **strategic DDD is the more important half**.

### 4.4 Real-World Example: E-commerce
| Subdomain | Bounded Context | Owns |
| ----- | ----- | ----- |
| **Catalog** | Catalog Service | Products, categories |
| **Pricing** | Pricing Service | Prices, discounts, taxes |
| **Order** | Order Service | Orders, order lifecycle |
| **Payment** | Payment Service | Transactions, refunds |
| **Inventory** | Inventory Service | Stock, reservations |
| **Shipping** | Shipping Service | Carriers, tracking |
| **Notification** | Notification Service | Emails, SMS |
Notice `**Product**`** means different things in different contexts**:

- In Catalog → name, description, images.
- In Inventory → SKU, quantity, warehouse.
- In Pricing → list price, discount rules.
That's exactly what DDD captures.

### 4.5 Senior Talking Point
>  _"DDD is not about classes and design patterns — it's about discovering the right boundaries for autonomy. If your teams can't describe the system using the same words as the business, you've drawn the wrong boundaries."_ 

---

## 5. Bounded Context
### 5.1 Definition
A **Bounded Context** is an explicit boundary within which a **specific domain model is consistent and unambiguous**.

The same term may mean different things in different bounded contexts — and that's fine, as long as the boundary is explicit.

### 5.2 Why It Matters for Microservices
>  A microservice's **boundary should align with a bounded context**. 

This produces services that:

- Have clear ownership of a model.
- Don't leak concepts.
- Can evolve independently.
### 5.3 Example: "Customer"
| Context | What "Customer" means |
| ----- | ----- |
| **Sales** | Lead, prospect, deal pipeline |
| **Billing** | Account holder, payment method, invoices |
| **Support** | Ticket creator, contact history |
| **Marketing** | Segments, campaigns, preferences |
If you force one `Customer` table for all four, every change to it requires coordination across teams. With bounded contexts, each team owns its model.

### 5.4 Context Mapping
Bounded contexts must still **integrate**. DDD defines relationships:

| Pattern | Meaning |
| ----- | ----- |
| **Shared Kernel** | Two contexts share a small common model (risky) |
| **Customer / Supplier** | Upstream/downstream, upstream provides what downstream needs |
| **Conformist** | Downstream uses upstream's model as-is |
| **Anti-Corruption Layer (ACL)** | Translation layer to keep external models from polluting yours |
| **Open Host Service** | A public, stable API for others to use |
| **Published Language** | An agreed-upon schema (e.g., events) |
### 5.5 Senior Talking Point
>  _"The unit of independent deployment is the bounded context, not the class, table, or REST endpoint. Whenever you can't change a service without coordinating with another team, you've crossed a bounded context boundary you shouldn't have."_ 

---

## 6. Single Responsibility Principle (SRP) at Service Level
### 6.1 SRP Recap
Originally from SOLID (Robert C. Martin):

>  _"A class should have only one reason to change."_ 

Applied to microservices:

>  _"A service should have only one reason to change — driven by a single business capability."_ 

### 6.2 Real-World Example
❌ **Bad:** `UserService` that handles authentication, profile management, preferences, notifications, audit logging.

✅ **Good:**

- `AuthService`  → auth & tokens.
- `ProfileService`  → user data.
- `NotificationService`  → emails/SMS.
- `AuditService`  → logs.
### 6.3 How to Tell You've Violated SRP
- One feature request touches multiple "responsibilities" of the same service.
- Multiple teams want to deploy the same service for different reasons.
- The service has wildly different scaling profiles internally.
- The PR description says "and also" twice.
### 6.4 Senior Talking Point
>  _"SRP at the service level is really about _**_one reason to redeploy_**_. If two unrelated features force the same redeploy cadence, you've coupled what shouldn't be coupled."_ 

---

## 7. Database per Service
### 7.1 The Rule
>  Each microservice owns its **own database/schema**. No other service may access it directly — only through its API. 

```
┌──────────────┐      ┌──────────────┐
│ Order Service│      │User Service  │
│   ┌──────┐   │      │   ┌──────┐   │
│   │Order │   │      │   │User  │   │
│   │ DB   │   │      │   │  DB  │   │
│   └──────┘   │      │   └──────┘   │
└──────────────┘      └──────────────┘
```
### 7.2 Why?
| Benefit | Explanation |
| ----- | ----- |
| **Loose coupling** | Schema changes don't break other services |
| **Polyglot persistence** | Right DB for the right job (PG, Mongo, Cassandra) |
| **Independent scaling** | Order DB can be partitioned differently than User DB |
| **Failure isolation** | One DB outage doesn't take down everything |
| **Clear ownership** | One team owns the schema |
### 7.3 Real-World Example: Uber
Uber's services use a mix:

- Trip data → custom datastore (Schemaless)
- Geo data → Cassandra
- Driver/rider profiles → MySQL
- Analytics → Pinot, Hive
Each service chooses what's appropriate for its workload.

### 7.4 Cross-Service Data Challenges
Once you split the DB, certain things get harder:

- **Joins** across services → must use API composition or CQRS.
- **Transactions** across services → use **Saga** (orchestrated or choreographed).
- **Reporting** → use a separate **data warehouse** populated via events/CDC.
### 7.5 Senior Talking Point
>  _"_`_Database per service_`_ isn't a performance optimization — it's a coupling boundary. The moment two services share a schema, you've effectively merged them; you just haven't admitted it yet."_ 

---

## 8. Shared Database Anti-Pattern
### 8.1 What it is
Multiple services read/write the **same database tables**.

```
┌──────────┐    ┌──────────┐    ┌──────────┐
│ Service A│    │ Service B│    │ Service C│
└────┬─────┘    └────┬─────┘    └────┬─────┘
     └────────────┬──┴──────────┬────┘
                  ▼             ▼
              ┌──────────────────┐
              │   Shared DB     │  ← BAD
              └──────────────────┘
```
### 8.2 Why It's an Anti-Pattern
- **Schema changes break everyone** — coordination nightmare.
- **Hidden coupling** — services depend on each other implicitly.
- **No clear ownership** — who can change what?
- **Locking and contention** at the DB level.
- **Defeats the whole point of microservices** — you have a distributed monolith.
### 8.3 When It Looks Acceptable (but isn't)
- "We're in a hurry, we'll fix it later" — you won't.
- "It's read-only for service B" — schema changes still break B.
- "Just one shared table" — that's how it starts.
### 8.4 How to Fix
- **API access** — service A asks service B's API for the data.
- **Event-driven replication** — service B publishes events; service A maintains its own read model.
- **CQRS** — separate read/write models.
- **Materialized views** — published by the owning service.
### 8.5 Senior Talking Point
>  _"A shared database is the gravitational singularity of a microservices architecture — once you have it, every service collapses back into the monolith from which you came."_ 

---

## 9. Stateless Services
### 9.1 What it Means
A **stateless service** does not retain client-specific state between requests. Each request contains all the info needed to process it.

- No in-memory sessions.
- No client-specific data in the process.
- State (if any) is in **external stores** (DB, Redis, Kafka).
### 9.2 Why Stateless Matters in Microservices
| Benefit | Explanation |
| ----- | ----- |
| **Horizontal scaling** | Any instance can serve any request |
| **Easy load balancing** | No sticky sessions required |
| **Resilience** | Crashing a pod loses no client state |
| **Rolling deployments** | Drain & replace instances freely |
| **Auto-scaling** | Kubernetes can scale up/down without complications |
### 9.3 Where Does State Go?
| Type of state | Where to keep it |
| ----- | ----- |
| **Session** | Redis, JWT (client-side) |
| **Domain state** | Database |
| **Cache** | Redis / Memcached |
| **Files** | S3 / GCS / Blob storage |
| **Workflow state** | Saga store / Temporal / Step Functions |
### 9.4 Real-World Example: Netflix Edge
Netflix's edge services are completely stateless. Any of the thousands of instances behind their load balancer can handle any user's request because session state lives in **EVCache (memcached fork)**.

### 9.5 Stateful Services Aren't Forbidden
Some workloads must be stateful:

- Kafka brokers.
- Cassandra nodes.
- ML training jobs.
For these, use **StatefulSets** (Kubernetes), with stable identities and persistent volumes.

### 9.6 Senior Talking Point
>  _"Statelessness is what gives microservices their elasticity. The moment a single instance can't be killed without losing data, you've lost cattle-style scaling and re-entered the world of pet servers."_ 

---

## 10. Service Granularity
### 10.1 The Goldilocks Problem
- Too **coarse** → mini-monolith, hard to deploy independently.
- Too **fine** → "nano-services", network overhead, ops nightmare, distributed transactions everywhere.
### 10.2 How to Find the Right Size
Indicators of **right size**:

- Owned end-to-end by one team.
- Can be rewritten in 2–4 weeks.
- Aligned with a bounded context / business capability.
- Has a small, stable API surface.
- Database/schema fits one team's mental model.
Indicators of **too small**:

- Two services always change together.
- Most "logic" is HTTP calls between services.
- Tests require spinning up 8 services.
Indicators of **too big**:

- Multiple teams contributing.
- Different parts have different scaling needs.
- Long deploy cycles, big release notes.
### 10.3 Real-World Example: Amazon
Amazon's famous **"2-pizza team"** rule: a team should be small enough to feed with two pizzas. Each such team owns one (or a few) services end-to-end.

### 10.4 Senior Talking Point
>  _"Granularity is not measured in lines of code; it's measured in coordination cost. The right service is the one a single team can own, deploy, and operate without needing meetings with other teams."_ 

---

## 11. Loose Coupling
### 11.1 Definition
**Loose coupling** means a service knows as little as possible about other services' internals — schemas, languages, deploy times, runtime.

### 11.2 Forms of Coupling (in order of badness)
| Type | Example | Severity |
| ----- | ----- | ----- |
| **Content coupling** | Service A reads B's DB | ❌❌❌ |
| **Temporal coupling** | A must be up when B calls it (synchronous) | ❌❌ |
| **Common coupling** | Shared mutable global state | ❌❌❌ |
| **Schema coupling** | A breaks if B changes its DTO | ❌ |
| **Behavioral coupling** | A depends on B's internal logic | ❌❌ |
### 11.3 How to Achieve Loose Coupling
- **API-only access** to other services.
- **Versioned, backward-compatible APIs**.
- **Async messaging / events** where possible.
- **Tolerant readers** — accept extra fields gracefully.
- **Independent deployments** — never deploy two services together as a batch.
### 11.4 Real-World Example
LinkedIn moves data between services via **Kafka events** rather than direct API calls. Producers don't know who's listening; consumers don't know who's producing. This is **temporal decoupling**.

### 11.5 Senior Talking Point
>  _"Loose coupling is what allows independent deployment. Every time two services need to be deployed together to avoid breakage, you've installed a hidden chain between them."_ 

---

## 12. High Cohesion
### 12.1 Definition
**Cohesion** = how related the functionality inside one service is.

A service is **highly cohesive** when all its code, data, and APIs serve **one well-defined purpose**.

### 12.2 Coupling vs Cohesion (the eternal pair)
>  **Good design = High cohesion + Loose coupling.** 

|  | Low Cohesion | High Cohesion |
| ----- | ----- | ----- |
| **Tight Coupling** | Spaghetti distributed monolith | Painful (over-coupled) |
| **Loose Coupling** | Pointless services | ✅ Ideal microservices |
### 12.3 Signs of Low Cohesion
- "OrderService" also handles emails and reporting.
- Service has unrelated tables in its DB.
- Multiple teams contribute for unrelated reasons.
- Different change cadences for different features.
### 12.4 Real-World Example
Uber once had a service called `dispatch` that handled trip matching, billing calculations, and driver notifications. As it grew, they split it into:

- `MatchingService` 
- `PricingService` 
- `NotificationService` 
Each had a clear, cohesive purpose.

### 12.5 Senior Talking Point
>  _"Cohesion is the inverse measure of why a service should be split. When you can't describe a service in a single sentence without saying 'and,' it's probably two services."_ 

---

## 13. API-first Design
### 13.1 What It Is
Design the **API contract first** — usually in **OpenAPI / Protobuf / GraphQL SDL** — before writing any code. The contract is the source of truth.

### 13.2 Why It Matters
- **Parallel work** — frontend, backend, mobile, partners all start at once.
- **Mocking / stubs** — clients can build against fake servers immediately.
- **Versioning** — explicit, deliberate API evolution.
- **Documentation comes for free** (Swagger UI, Redoc).
- **Contract testing** (Pact, Spring Cloud Contract).
- **Code generation** for clients and servers.
### 13.3 Workflow
1. Discuss capability with stakeholders.
2. Draft OpenAPI spec.
3. Review with consumers.
4. Generate mock server + client SDKs.
5. Implement the actual service.
6. CI validates the implementation against the spec.
### 13.4 Real-World Example: Stripe
Stripe is famous for its **API-first culture**. Their public APIs are designed with extreme care — backward compatibility is sacred, versioning is explicit (date-based: `2023-10-16`), and every change is reviewed for impact on integrators.

### 13.5 Senior Talking Point
>  _"Treat your API as a product, not as an implementation detail. The contract is the boundary; everything else is replaceable."_ 

---

## 14. Backend for Frontend (BFF)
### 14.1 The Problem
Different clients (web, iOS, Android, smart TV, partners) have **different needs**:

- Mobile wants small payloads (slow networks).
- Web wants rich data with images.
- TV wants pre-formatted strings.
- Partners want stable, contract-versioned APIs.
If one backend serves them all, it becomes a mess of conditional logic.

### 14.2 BFF Pattern
Build a **dedicated backend per client type** — each tailored to that client's needs.

```
┌──────┐     ┌──────┐     ┌──────┐
│ Web  │     │ iOS  │     │ TV   │
└──┬───┘     └──┬───┘     └──┬───┘
   ▼            ▼            ▼
┌──────┐    ┌──────┐    ┌──────┐
│WebBFF│    │iOSBFF│    │TvBFF │
└──┬───┘    └──┬───┘    └──┬───┘
   └────────────┼────────────┘
                ▼
      ┌─────────────────────┐
      │  Domain microservices│
      │ (Order, Catalog, ...)│
      └─────────────────────┘
```
### 14.3 Responsibilities of a BFF
- Aggregate calls to multiple downstream services.
- Reshape data to match the client.
- Cache for the client.
- Handle client-specific auth.
- Hide internal service topology.
### 14.4 Real-World Example: Netflix
Netflix pioneered the BFF pattern (they called it the **"Edge Service"** layer). Each device type has its own edge tailored to its bandwidth and UI needs.

SoundCloud also publicized the pattern under the BFF name.

### 14.5 Pitfalls
- **BFFs become too thick** — start hosting business logic that should be in domain services.
- **Duplication** — three BFFs doing similar things.
- **Ownership** — usually owned by the **client team**, not a separate backend team.
### 14.6 Senior Talking Point
>  _"A BFF is owned by the team that owns the experience. It exists to optimize the client, not to be another shared service. The moment all BFFs converge, you've reinvented the API gateway."_ 

---

## 15. Micro Frontends
### 15.1 What They Are
Apply the microservices philosophy to the **frontend**: split a UI into independently developed, deployed, and owned pieces — each owned by the team that owns the corresponding backend.

### 15.2 Why?
Without micro frontends:

- Backend is split into 50 microservices, but the **frontend is still one monolithic SPA**.
- One team's UI change blocks everyone else.
- The frontend becomes the new bottleneck.
### 15.3 Implementation Approaches
| Approach | Description |
| ----- | ----- |
| **Build-time integration** | Each team publishes an npm package; shell app composes them at build time |
| **Server-side composition** | Edge server stitches HTML from multiple services (Edge-Side Includes) |
| **Run-time via iframes** | Simple, isolated, but ugly UX |
| **Run-time via Web Components / Module Federation** | Modern approach (Webpack 5 Module Federation, single-spa) |
### 15.4 Real-World Examples
- **Spotify** — each "tab" in the desktop app is a separate micro frontend.
- **IKEA** — different sections of ikea.com are owned by different teams.
- **Zalando** — leading public advocate; uses their own framework "Mosaic".
### 15.5 Trade-offs
✅ Team autonomy, independent deploys, polyglot frameworks possible.
❌ Bundle size, performance overhead, design consistency challenges, shared state hard.

### 15.6 Senior Talking Point
>  _"Microservices without micro frontends often just move the bottleneck to the UI team. End-to-end vertical ownership — UI + API + DB — is what truly enables autonomous teams."_ 

---

## Putting It All Together — Senior-Level Mental Model
```
Business Domain
   │
   ▼
[Subdomains via DDD]
   │
   ▼
[Bounded Contexts]
   │
   ▼
[Microservices (1:1 with bounded contexts)]
   │
   ├── Database per service (no shared DB)
   ├── Stateless (state externalized)
   ├── High cohesion, loose coupling
   ├── API-first contract (OpenAPI/Protobuf)
   ├── Right granularity (2-pizza team owns it)
   └── Resilient (timeouts, retries, breakers, bulkheads)
   │
   ▼
[BFFs for each client experience]
   │
   ▼
[Micro Frontends for UI autonomy]
```
---

## Top Interview Questions (with Quick-Answer Pointers)
1. **"Why split a monolith?"** → Independent deployment + team autonomy, not "scale" alone.
2. **"What are the costs of microservices?"** → Network, ops, distributed data, observability, transactional complexity.
3. **"How do you decide service boundaries?"** → DDD bounded contexts + Conway's Law + 2-pizza ownership.
4. **"Why database per service?"** → Coupling boundary, not just performance.
5. **"How do you handle distributed transactions?"** → Sagas + idempotency + eventual consistency.
6. **"What's the difference between BFF and API Gateway?"** → BFF is client-specific; gateway is cross-cutting (auth, routing, rate-limit).
7. **"How do you ensure loose coupling?"** → API-only access, async events, versioning, tolerant readers.
8. **"What is the strangler fig pattern?"** → Incrementally replace monolith via proxy + new services.
9. **"What is CAP theorem in practice?"** → CP vs AP — and remember PACELC.
10. **"How do you avoid distributed monolith?"** → Independent deploys, no shared schema, async wherever possible.






<!--- Eraser file: https://app.eraser.io/workspace/XtalLcp56TqNdwObjDOz --->