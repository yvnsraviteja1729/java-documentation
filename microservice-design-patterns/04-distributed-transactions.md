<p><a target="_blank" href="https://app.eraser.io/workspace/tYKaS4r43DPlcpNQdrAG" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>

In a distributed system, a single business operation often spans multiple services, each with its own database. You can't use a single ACID transaction across them. These patterns are how the industry solves that problem.

---

# Part 1: The Core Problem
A monolith with one database can do this:

```sql
BEGIN;
  UPDATE accounts SET balance = balance - 100 WHERE id = 'A';
  UPDATE accounts SET balance = balance + 100 WHERE id = 'B';
COMMIT;  -- atomic, isolated, durable
```
A microservices architecture **cannot**:

```
PaymentService.debit('A', 100)    ← own DB
OrderService.create(...)          ← own DB
ShippingService.schedule(...)     ← own DB
```
If step 2 fails after step 1 succeeded, you have **inconsistent state**. The patterns below address this in different ways.

There are **two philosophies**:

1. **Strong consistency** — actually try to coordinate atomic commit across services (2PC, 3PC). Powerful but blocking and fragile.
2. **Eventual consistency** — accept temporary inconsistency, use messaging + compensation (Saga, Outbox/Inbox, Idempotency). The modern microservices default.
---

# Part 2: Saga Pattern
A **saga** is a sequence of local transactions, where each step publishes an event/message that triggers the next. If a step fails, previously completed steps are undone via **compensating transactions**.

There's no global lock, no global commit — just a chain of local commits with rollback handlers.

```
Order Saga:
1. CreateOrder        ← if fails: stop
2. ReserveInventory   ← if fails: cancel order
3. ChargePayment      ← if fails: release inventory + cancel order
4. ScheduleShipping   ← if fails: refund + release inventory + cancel order
```
Sagas come in two flavors: **choreography** and **orchestration**.

## 2.1 Choreography-Based Saga
**No central coordinator.** Each service listens for events and emits its own. The "workflow" is implicit in who subscribes to what.

```
OrderService     → publishes OrderCreated
                                  │
PaymentService   → consumes OrderCreated
                 → publishes PaymentCharged or PaymentFailed
                                  │
InventoryService → consumes PaymentCharged
                 → publishes InventoryReserved or InventoryFailed
                                  │
ShippingService  → consumes InventoryReserved
                 → publishes ShipmentScheduled
```
Compensation works the same way — failure events trigger reverse handlers:

```
PaymentFailed  → OrderService cancels order
InventoryFailed → PaymentService refunds + OrderService cancels order
```
### Pros
- **Fully decoupled** — no central authority, services know only about events.
- **Simple to start** — just publish and subscribe.
- **Resilient** — no single point of failure.
- **Scales naturally** — each service handles its own concerns.
### Cons
- **Hard to understand the overall flow** — it's emergent, not written down anywhere.
- **Cyclic dependencies** are easy to introduce accidentally.
- **Hard to debug** — workflow is spread across logs of N services.
- **Hard to change** — adding a new step means editing multiple services.
- **Risk of "event soup"** as the system grows.
### When to use
- Small number of steps (3–4).
- Steps are loosely related; teams own them independently.
- You don't need explicit visibility of workflow state.
## 2.2 Orchestration-Based Saga
A **central orchestrator** (a workflow engine, or a stateful service) explicitly drives the saga: it sends commands to each service and reacts to their responses.

```
┌───────────────────┐
       │  Order Saga       │
       │  Orchestrator     │
       └─────┬─────────────┘
             │
   ┌─────────┼─────────┬───────────┐
   ▼         ▼         ▼           ▼
Payment   Inventory  Shipping   Notification
```
The orchestrator is a state machine:

```
START → CHARGE_PAYMENT → RESERVE_INVENTORY → SCHEDULE_SHIPPING → DONE
    ↓ fail              ↓ fail              ↓ fail
CANCEL_ORDER     REFUND + CANCEL    REFUND + RELEASE + CANCEL
```
Implemented with tools like **Temporal, Camunda, AWS Step Functions, Netflix Conductor**, or hand-rolled state machines.

### Pros
- **Workflow is explicit, centralized, visible** — easy to read, audit, modify.
- **Easier to add/remove steps**.
- **Easier debugging** — orchestrator has the full state.
- **Easier compensation logic** — centralized.
- **Easier to handle timeouts, retries, conditional branches**.
### Cons
- **Orchestrator can become a god service** — too much business logic in one place.
- **Single point of failure** (mitigated by clustering).
- **Couples services to the orchestrator's commands** (less to each other, but still coupled).
- **More infrastructure** (a workflow engine).
### When to use
- Complex workflows (5+ steps), conditional branches, parallel tasks.
- Need clear visibility, monitoring, and ability to manually intervene (replay, skip, retry).
- Long-running processes (days/weeks).
### Quick comparison
|  | Choreography | Orchestration |
| ----- | ----- | ----- |
| Coordinator | None (events) | Central engine |
| Coupling | Loose | Hub-and-spoke |
| Visibility | Low | High |
| Best for | Simple flows | Complex flows |
| Risk | Spaghetti | God service |
---

# Part 3: Two-Phase Commit (2PC)
A classic distributed transaction protocol from the database world. A **transaction coordinator** asks all participants to "prepare" then "commit."

### Phases
**Phase 1 — Prepare:**

```
Coordinator → "Can you commit?" → Participant A
Coordinator → "Can you commit?" → Participant B
Coordinator → "Can you commit?" → Participant C

Each participant:
  - locks resources
  - writes to the WAL
  - replies YES or NO
```
**Phase 2 — Commit (or Abort):**

```
If all said YES → Coordinator: "COMMIT" → all participants commit
If any said NO  → Coordinator: "ABORT"  → all participants roll back
```
### Pros
- **Strong ACID guarantees** across multiple resources.
- Conceptually simple.
- Supported by XA-compliant databases & transaction managers (JTA in Java).
### Cons (why it's avoided in microservices)
- **Blocking protocol** — participants hold locks during the entire prepare phase. Other transactions wait.
- **Coordinator is a single point of failure** — if coordinator crashes after prepare but before commit/abort, participants are stuck holding locks (the "in-doubt" state).
- **Slow** — multiple network round trips while holding locks.
- **Requires all participants to support 2PC / XA** — most modern services and queues don't.
- **Doesn't scale** — every additional participant increases latency and lock contention.
- **Tightly couples services** to a coordinator and to each other's availability.
### When 2PC is OK
- Within a **single datacenter** across XA-compliant databases (legacy banking systems, ESBs).
- **Short-lived transactions** between **few** participants.
- When **strong consistency is non-negotiable** and you accept the latency/availability cost.
In modern microservices, **almost always prefer Sagas + Outbox + Idempotency** over 2PC.

---

# Part 4: Three-Phase Commit (3PC)
An extension of 2PC that tries to remove the blocking problem by adding a third phase.

### Phases
1. **CanCommit?** — coordinator asks if participants can commit.
2. **PreCommit** — if all said yes, coordinator says "prepare to commit." Participants acknowledge.
3. **DoCommit** — coordinator says "commit now."
The key idea: by the time anyone commits, everyone knows the outcome will be commit. If the coordinator dies, participants can timeout and **decide to commit on their own** (because they all received PreCommit).

### Pros
- **Non-blocking** in the absence of network partitions.
- Solves the "in-doubt" problem of 2PC.
### Cons
- **Doesn't handle network partitions correctly** — under split-brain, different sides can decide differently → inconsistency.
- **Even more network round trips** (slower than 2PC).
- **Rarely used in practice** — too complex, too slow, doesn't actually give you the guarantees you want under realistic failure modes.
- **Modern systems use consensus algorithms (Paxos, Raft)** instead, which handle partitions correctly.
### When to use
- Almost never. 3PC is mostly of academic interest. If you need partition-tolerant consistent commit, use **Paxos/Raft** (e.g., etcd, Spanner, CockroachDB).
---

# Part 5: Compensating Transactions
A **compensating transaction is the semantic undo** of a previously committed local transaction. It's the building block of sagas.

You **cannot rollback** a committed transaction in a different database, so you do the next best thing: **execute a new transaction that logically reverses it**.

### Examples
| Original Action | Compensation |
| ----- | ----- |
| Charge $100 to card | Refund $100 |
| Reserve seat 14A | Release seat 14A |
| Send confirmation email | Send "ignore previous email" |
| Reduce inventory by 5 | Increase inventory by 5 |
| Create shipment | Cancel shipment |
### Important properties
- **Compensations must be idempotent** — they may be retried.
- **Compensations are not always perfect inverses** — sending an email can't be unsent; you can only send a correction. This is called a "real-world side effect" problem.
- **Order matters** — typically compensate in **reverse order** of execution.
- **Some operations are not compensable** — design the saga so non-compensable steps come **last**.
- **Semantic, not transactional** — the compensating action happens in its own local transaction, with its own visibility.
### Example
```
Saga steps:
  T1: ReserveInventory(orderId, qty=5)
  T2: ChargePayment(orderId, $100)
  T3: ScheduleShipping(orderId)
If T3 fails:
  C2: RefundPayment(orderId, $100)
  C1: ReleaseInventory(orderId, qty=5)
```
---

# Part 6: Outbox Pattern
**Problem:** A service needs to atomically (a) update its database AND (b) publish an event. If it does both separately, one can succeed and the other fail → inconsistency.

```
service.updateDB();             ← committed
broker.publish(event);          ← network fails! event lost
```
Or vice versa:

```
broker.publish(event);          ← published
service.updateDB();             ← crashes! event sent for non-existent state
```
This is the **dual-write problem**.

### Solution: Outbox Pattern
Write the event to an `**outbox**`** table in the same database transaction** as your business state change.

```sql
BEGIN;
  INSERT INTO orders (id, status) VALUES (123, 'PLACED');
  INSERT INTO outbox (id, topic, payload) VALUES (uuid(), 'orders', '{...}');
COMMIT;
```
A **separate process (relay/CDC)** reads the outbox and publishes to Kafka/RabbitMQ:

```
┌──────────────┐                ┌────────────┐
│ Service DB   │                │  Broker    │
│  ┌─────────┐ │                │ (Kafka)    │
│  │ orders  │ │                └─────▲──────┘
│  ├─────────┤ │                      │
│  │ outbox  │ │   ┌──────────────┐   │
│  └────┬────┘ │   │ Outbox Relay │───┘
└───────┼──────┘   │ (Debezium /  │
        └─────────►│  poll worker)│
                   └──────────────┘
```
### Implementation options
- **Polling worker** — `SELECT * FROM outbox WHERE published=false`  periodically.
- **Change Data Capture (CDC)** — tools like **Debezium** tail the database WAL/binlog and push changes to Kafka. No polling, very low latency.
### Guarantees
- **At-least-once delivery** — combined with idempotent consumers, this is effectively-once.
- Atomicity is provided by the **local DB transaction** — no distributed transaction needed.
### Pros
- Solves dual-write reliably.
- No need for XA / 2PC.
- Simple, battle-tested pattern.
### Cons
- Slight latency (polling interval) unless using CDC.
- Outbox table grows; need a cleanup job.
- Requires the relay infrastructure.
**This is the standard pattern for reliable event publishing in microservices today.**

---

# Part 7: Inbox Pattern
The **mirror of the outbox**, applied to the consumer side.

**Problem:** A consumer receives a message, updates its DB, and acks the message. If it crashes after DB update but before ack, the message is redelivered → double processing. If it acks before DB update, a crash before update means lost message.

### Solution: Inbox Pattern
When a message arrives, in **one local transaction**:

1. Insert the message ID into an `inbox`  table (with a unique constraint).
2. Apply the business logic.
3. Commit.
```sql
BEGIN;
  INSERT INTO inbox (message_id) VALUES ('msg-123');   -- fails if duplicate
  UPDATE accounts SET balance = balance + 100 WHERE id = 'A';
COMMIT;
```
If the message is redelivered, the unique constraint fails, the transaction rolls back, and the message is silently dropped (already processed).

### Pros
- Provides **idempotent processing** with full transactional safety.
- Works with any at-least-once broker.
- Combined with the outbox pattern → **reliable end-to-end messaging** without distributed transactions.
### Cons
- Inbox table grows; needs cleanup with TTL.
- Slight overhead per message.
**Outbox + Inbox + at-least-once broker = effectively-once processing**, which is what most real systems aim for.

---

# Part 8: Transactional Messaging
An umbrella term for any technique that **atomically links a state change with sending/receiving a message**, solving the dual-write problem.

### Approaches
1. **Outbox + Inbox** — the modern, recommended approach (described above).
2. **Kafka Transactions** — Kafka's `transactional.id`  lets a producer atomically write to multiple partitions and commit consumer offsets in one transaction. Combined with `isolation.level=read_committed`  on consumers, gives **exactly-once semantics within Kafka**. Doesn't help if you also write to a DB.
3. **XA / 2PC across DB and Broker** — supported by some brokers (older ActiveMQ, IBM MQ + XA databases). Strong consistency but blocking, slow, and requires XA support everywhere. Avoided in modern systems.
4. **Listen-to-yourself / Event-First** — service publishes the event to itself (via the broker), consumes it, and only then updates its DB. Rarely used.
### Key insight
You can't have a single distributed transaction across a database and a message broker without 2PC. The **outbox/inbox pattern sidesteps the problem** by reducing it to two **local** transactions plus message redelivery and idempotency.

---

# Part 9: Idempotency
An operation is **idempotent** if executing it multiple times produces the same result as executing it once.

```
SET balance = 100        ← idempotent
balance = balance + 10   ← NOT idempotent
DELETE WHERE id = 5      ← idempotent
INSERT ...               ← NOT idempotent
```
### Why it matters
In distributed systems, **everything gets retried**: network failures, broker redelivery, client retries, gateway retries. Without idempotency, retries cause duplicate side effects.

### Techniques
1. **Idempotency keys** — client supplies a unique key per operation; server stores keys and refuses duplicates.Stripe, PayPal, AWS APIs all use this.POST /payments
Idempotency-Key: 7a3f...
2. **Unique constraints** — use natural or generated unique IDs and rely on `INSERT ... ON CONFLICT DO NOTHING`  / unique index violations.
3. **Conditional updates** — `UPDATE ... WHERE version = X`  (optimistic locking).
4. **Inbox table** — store processed message IDs; refuse duplicates.
5. **Naturally idempotent operations** — design APIs around `set`  / `upsert`  / `assign`  rather than `add`  / `increment` .
6. **Deduplication windows** — hash payloads, store hashes with TTL.
### The golden rule
>  **Make your consumers idempotent.** It's cheaper, simpler, and more reliable than chasing exactly-once delivery from the infrastructure. 

---

# Part 10: Delivery Guarantees
These describe what the messaging infrastructure (broker + producer + consumer) promises about message delivery.

## 10.1 At-Most-Once Delivery
"**Send it, but it's OK to lose it.**"

- Producer fires and forgets. Consumer doesn't ack reliably.
- Each message is delivered **0 or 1 times** — no duplicates, but possible loss.
### Examples
- UDP-style telemetry, metrics, logs where occasional loss is acceptable.
- Kafka with `acks=0` .
- Fire-and-forget HTTP calls without retries.
### Pros
- Lowest latency, highest throughput.
- Simplest to implement.
### Cons
- Data loss is possible.
- Unacceptable for business-critical operations.
### When to use
- Metrics, telemetry, click events, logs at high volume where you tolerate sampling loss.
## 10.2 At-Least-Once Delivery
"**Send it until you know it was received.**"

- Producer retries on failure. Consumer acks after processing.
- Each message is delivered **1 or more times** — no loss, but possible duplicates.
### Examples
- Default for almost every modern broker: Kafka, RabbitMQ, SQS, Pub/Sub.
- HTTP POST with retries on 5xx.
### Pros
- No data loss.
- Simple to implement.
- Works with all brokers.
### Cons
- **Duplicates are inevitable** — consumers must be **idempotent**.
### When to use
- **The default for almost everything.** Combine with idempotent consumers and you get effectively-once processing.
## 10.3 Exactly-Once Processing
"**Each message has its effect applied exactly once.**"

This is what users actually want. It's distinct from "exactly-once delivery" (which is theoretically impossible across systems — see the Two Generals Problem).

### How it's achieved (effectively-once)
Combination of:

- **At-least-once delivery** (broker level).
- **Idempotent consumers** (application level — inbox pattern, idempotency keys, unique IDs).
Result: messages may be delivered multiple times, but only **applied once**.

### Special case: Kafka Transactions
For **read-process-write pipelines entirely within Kafka** (input topic → processing → output topic), Kafka offers true exactly-once semantics using transactions:

```java
producer.initTransactions();
producer.beginTransaction();
producer.send(...);
producer.sendOffsetsToTransaction(offsets, consumerGroupMetadata);
producer.commitTransaction();
```
Consumer with `isolation.level=read_committed` only sees committed records.

This works because Kafka controls both endpoints. As soon as you involve an external DB or HTTP call, you're back to **at-least-once + idempotency**.

### Should you chase exactly-once?
**Usually no.** It's:

- More expensive (transactions add latency).
- More complex.
- Limited in scope (only within the controlling system).
**Better strategy:** at-least-once + idempotent consumers. Simpler, faster, equivalent in practice.

## 10.4 Comparison
| Guarantee | Loss? | Duplicates? | Throughput | Complexity | Use Case |
| ----- | ----- | ----- | ----- | ----- | ----- |
| At-most-once | ✅ Possible | ❌ No | Highest | Lowest | Metrics, telemetry |
| At-least-once | ❌ No | ✅ Possible | High | Medium | <p>**Default**</p><p>, with idempotency</p> |
| Exactly-once (effectively) | ❌ No | ❌ No | Medium | Medium-High | Payments, billing, anything financial |
| Exactly-once (Kafka transactions) | ❌ No | ❌ No | Lower | High | Stream processing pipelines within Kafka |
---

# Part 11: How These Patterns Fit Together
A real-world e-commerce checkout, using **all of them together**:

```
1. POST /checkout (Idempotency-Key: abc123)
        │
        ▼
2. OrderService:
     BEGIN TX
       INSERT order
       INSERT outbox (event: OrderPlaced)
     COMMIT
        │
        ▼
3. Outbox Relay (Debezium) → Kafka topic "orders"
        │
        ▼
4. PaymentService consumes (at-least-once):
     BEGIN TX
       INSERT inbox (msg_id) -- idempotency
       Charge card
       INSERT outbox (event: PaymentCharged or PaymentFailed)
     COMMIT
        │
        ├─ PaymentCharged → InventoryService → ShippingService (saga continues)
        │
        └─ PaymentFailed → triggers compensations (cancel order)
```
**Pattern stack:**

- **Idempotency key** at the API layer → safe client retries.
- **Outbox** at the producer side → reliable event publishing.
- **Inbox** at the consumer side → idempotent processing.
- **At-least-once** delivery from Kafka → no data loss.
- **Saga (choreography or orchestration)** → multi-service workflow.
- **Compensating transactions** → rollback on failure.
- **No 2PC anywhere** → loosely coupled, scalable, partition-tolerant.
---

# Part 12: Decision Guide
| Need | Pattern |
| ----- | ----- |
| Multi-service workflow with eventual consistency | <p>**Saga**</p><p> (choreography for simple, orchestration for complex)</p> |
| Reliable event publishing from a service | <p>**Outbox**</p><p> (with Debezium / CDC)</p> |
| Idempotent message processing | <p>**Inbox**</p><p> + idempotency keys</p> |
| Undo a previous step in a saga | **Compensating transaction** |
| Strong ACID across services (rare, legacy) | <p>**2PC / XA**</p><p> — only if absolutely required</p> |
| Exactly-once within Kafka pipeline | **Kafka transactions** |
| Exactly-once across heterogeneous systems | <p>**At-least-once + idempotent consumer**</p><p> (effectively-once)</p> |
| Safe client retries on HTTP APIs | **Idempotency-Key header** |
| High-volume, lossy-tolerant data | **At-most-once** |
---

# TL;DR
- **Sagas + compensations** are how modern microservices coordinate distributed work — choose **choreography** for simple flows, **orchestration** (Temporal/Camunda) for complex ones.
- **2PC and 3PC** give strong consistency but block, scale poorly, and are mostly avoided. Use them only inside one trust boundary if you must.
- **Outbox + Inbox** solve the dual-write problem cleanly, using only local transactions plus messaging — no distributed coordination needed.
- **Idempotency** is the most important property of consumers in any distributed system; without it, retries break things.
- **At-least-once delivery + idempotent consumers = effectively-once processing**, and that's what almost every production system actually does. Don't chase true "exactly-once" — make consumers idempotent instead.




<!--- Eraser file: https://app.eraser.io/workspace/tYKaS4r43DPlcpNQdrAG --->