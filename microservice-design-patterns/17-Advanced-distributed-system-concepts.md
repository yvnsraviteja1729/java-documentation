<p><a target="_blank" href="https://app.eraser.io/workspace/npnEbn0SAGt4lNd2T22e" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>

The deep theoretical and practical foundations that explain **why distributed systems behave the way they do** — and why some "simple" features are surprisingly hard.

---

# Part 1: The Two Big Theorems
These two theorems shape almost every architectural decision in distributed databases and systems.

## 1.1 CAP Theorem
Formulated by Eric Brewer (2000), proven by Gilbert & Lynch (2002).

>  In a distributed system, you can have **at most two** of these three properties at any given time:  

### The realistic interpretation
Network partitions **will happen** in any real distributed system (packet loss, switch failures, datacenter outages). So **P is non-negotiable**. The actual choice is between **C and A during a partition**:

- **CP system** — when a partition occurs, refuse to serve requests on the minority side to preserve consistency. Examples: HBase, MongoDB (with majority writes), etcd, Zookeeper, Consul.
- **AP system** — when a partition occurs, keep accepting reads/writes everywhere; reconcile later. Examples: Cassandra (default tunable), DynamoDB, Riak, CouchDB.
### The misconception
"CAP says you must give up one of the three." Reality: you only sacrifice C or A **during a partition**. Most of the time the network is fine and you get all three.

### Trade-off in practice
| You want… | Pick |
| ----- | ----- |
| Bank balances, inventory, order systems | <p>**CP**</p><p> (refuse on partition is safer)</p> |
| Shopping carts, social timelines, sensor data | <p>**AP**</p><p> (always accept; reconcile later)</p> |
## 1.2 PACELC Theorem
Daniel Abadi (2010) — extends CAP to address what happens **when there is no partition**:

>  If there is a **P**artition: choose between **A**vailability and **C**onsistency.
**E**lse (normal operation): choose between **L**atency and **C**onsistency. 

### Why this matters
CAP only describes failure conditions. PACELC captures the everyday trade-off: even with a healthy network, **strong consistency requires coordination**, which adds latency. Eventual consistency is faster.

### Examples
| System | Partition behavior | Normal behavior |
| ----- | ----- | ----- |
| Cassandra, DynamoDB, Riak | <p>**A**</p><p> (stay available)</p> | <p>**L**</p><p> (low latency) → PA/EL</p> |
| HBase, BigTable | **C** | <p>**C**</p><p> → PC/EC</p> |
| MongoDB (majority writes) | **C** | <p>**C**</p><p> → PC/EC</p> |
| Spanner | **C** | <p>**C**</p><p> (uses TrueTime to minimize latency cost) → PC/EC</p> |
| MySQL with sync replication | **C** | <p>**C**</p><p> → PC/EC</p> |
### The deeper insight
"AP" databases like Cassandra and DynamoDB are AP **and** EL — they're optimized for low latency normally, and stay available during failures. That's a coherent design. "CP" systems are usually also EC — they pay the latency cost always, partition or not.

---

# Part 2: Consensus Algorithms
## 2.1 The Consensus Problem
Get a group of nodes to agree on a single value, even when:

- Messages can be lost or reordered.
- Nodes can crash and restart.
- Network can partition.
This is the foundation of **leader election, distributed locks, replicated state machines, distributed databases, and config stores**.

The **FLP Impossibility Result** (1985) proves that perfect consensus is impossible in a fully asynchronous system with even one faulty node. Real algorithms work around this with **timeouts** (which assume some degree of synchrony) and **majority voting (quorums)**.

## 2.2 Paxos
Proposed by Leslie Lamport (1989, properly published 1998 in "The Part-Time Parliament"). The original solution to consensus, foundational but **notoriously hard to understand and implement correctly**.

### Roles
- **Proposer** — proposes a value.
- **Acceptor** — votes on proposals.
- **Learner** — learns the chosen value.
### Two phases (Basic Paxos)
**Phase 1 — Prepare:**

- Proposer picks a proposal number `n`  and sends `prepare(n)`  to a majority of acceptors.
- Acceptor: if `n`  is higher than any prepare it's seen, promise not to accept lower-numbered proposals; respond with the highest-numbered proposal it has already accepted (if any).
**Phase 2 — Accept:**

- If proposer gets a majority of promises, it sends `accept(n, value)`  (using the value from any prior accepted proposal it learned about, else its own).
- Acceptors accept unless they've promised a higher `n` .
- Once a majority accepts, the value is **chosen**.
### Why it's hard
- Multiple proposers can stall progress (livelock).
- Real systems need **Multi-Paxos** (run consecutive instances for a sequence of values, with a stable leader) to be efficient.
- The paper's correctness proofs are dense; many published implementations are buggy.
### Where it's used
- Google Chubby (lock service).
- Google Spanner (Paxos groups per shard).
- Many homegrown systems at Google/Microsoft.
## 2.3 Raft
Designed by Ongaro & Ousterhout (2014) at Stanford with the explicit goal: **understandable consensus**. Equivalent power to Multi-Paxos but far easier to reason about and implement.

### Key ideas
- **Strong leader** — at any time, exactly one leader handles all client requests and replicates the log.
- **Decompose into three subproblems**: leader election, log replication, safety.
### Roles (state machine)
- **Follower** — passive; responds to the leader's heartbeats.
- **Candidate** — running for election.
- **Leader** — handles all writes, replicates log to followers.
### Leader election
- Each follower has a **randomized election timeout** (e.g., 150–300 ms).
- If no heartbeat from leader before timeout → becomes candidate, increments **term**, votes for itself, requests votes from others.
- Wins if it gets a majority of votes for its term → becomes leader.
- Sends heartbeats to suppress new elections.
- Randomized timeouts prevent split votes most of the time.
### Log replication
- Client sends command to leader → leader appends to its log → sends `AppendEntries`  RPCs to followers.
- Once a **majority** of followers have written it, leader **commits** the entry and applies it to its state machine; followers do too.
- Each entry is `(term, index, command)` .
### Safety guarantees
- A leader for term `T`  has all committed entries from prior terms (election restriction).
- Once committed, an entry is durable across all future leaders.
### Where it's used
- **etcd** (Kubernetes' brain).
- **Consul** (HashiCorp).
- **CockroachDB**.
- **TiKV / TiDB**.
- **MongoDB** (replica set protocol since 3.2 is Raft-like).
- **Kafka KRaft mode** (the new ZooKeeper-less metadata layer).
### Why Raft "won"
Equivalent guarantees to Paxos with vastly better explanability → most modern systems chose Raft.

## 2.4 Other Consensus Algorithms (briefly)
- **ZAB** (Zookeeper Atomic Broadcast) — ZooKeeper's protocol; predates Raft, similar principles.
- **PBFT, HotStuff** — Byzantine Fault Tolerant consensus (handles malicious nodes, not just crashes); used in some blockchains.
- **EPaxos, Flexible Paxos** — research variants for lower latency / different quorum trade-offs.
---

# Part 3: Quorum
## 3.1 What It Is
A **quorum** is the minimum number of nodes that must agree before an operation is considered successful. Quorums are how distributed systems get **strong guarantees from majority voting**, without contacting every node.

For an `N`-node cluster:

- **Write quorum (W)** — number of nodes that must acknowledge a write.
- **Read quorum (R)** — number of nodes that must respond to a read.
### Strong consistency rule
If `W + R > N`, then any read quorum is guaranteed to overlap with any write quorum, so reads see the latest write.

The most common choice is `W = R = ⌈(N+1)/2⌉` (a simple majority), so any two majorities intersect. With N=3, majority is 2; with N=5, majority is 3.

### Examples
- **Raft / Paxos** — require majority for elections and commits.
- **Kafka** — `min.insync.replicas`  controls write quorum.
- **Cassandra** — tunable per query: `ONE` , `QUORUM` , `LOCAL_QUORUM` , `ALL` .
- **DynamoDB** — quorum reads/writes internally (~configurable as eventual or strong).
## 3.2 Why Quorum, Not "All"
- **Availability** — tolerates `(N-1)/2`  failures while still making progress.
- **Latency** — wait for the majority, not the slowest node (one slow replica doesn't block).
## 3.3 Quorum vs Replication Factor
- **Replication factor (RF)** — how many copies exist.
- **Quorum** — how many of those copies must respond.
You can have RF=5 but `W=3, R=3` for performance, sacrificing some durability tolerance.

---

# Part 4: Leader Election
## 4.1 What It Is
The process by which one node in a cluster becomes the **single coordinator** for a task — typically writes, scheduling, or sequencing.

### Why a leader?
Many problems become much easier with a single leader:

- Only one node ordering writes → no concurrent conflict resolution.
- One node deciding work assignments → no double-assignment.
- One node generating IDs / timestamps → simple total ordering.
### How leaders are elected
- **Via consensus algorithm** (Raft, Paxos, ZAB) — the safest, used by etcd, Consul, ZooKeeper, Kafka KRaft, MongoDB.
- **Via a coordination service** (ZooKeeper, etcd) — clients race to create an ephemeral node; whoever succeeds is leader. Others watch for changes.
- **Via a distributed lock** — whoever holds the lock is leader (described below).
- **Via gossip + ranking** — pick the node with lowest ID among reachable nodes; simple but vulnerable to split brain.
### Failure handling
- Leader sends **heartbeats** to followers.
- If heartbeats stop (timeout), followers initiate a new election.
- **Term/epoch numbers** prevent old leaders from acting once deposed (a write from an outdated leader is rejected).
### Where it appears
- Kafka controller, MongoDB primary, Postgres patroni, Elasticsearch master, Kubernetes scheduler leader, distributed cron, Spark driver.
---

# Part 5: Distributed Locks
## 5.1 What It Is
A **mutex across multiple machines** — only one process across the cluster can hold the lock at a time. Used for:

- "Run this scheduled job in only one place."
- "Process this order exactly once."
- "Become the leader of this work."
- Coordinating access to a shared external resource.
## 5.2 Implementations
### Redis-based — `SET NX PX` 
```bash
SET lock:job:7 "owner-uuid" NX PX 30000   # acquire if not exists, 30s TTL
```
Release by checking the value first (Lua script for atomicity):

```lua
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
end
return 0
```
**Pitfalls:**

- Process pauses (GC, kernel lockup) → TTL expires → another process gets the lock → original process resumes thinking it still has it.
- Single-Redis isn't safe under failover (the new primary may not have replicated the lock yet).
### Redlock (Redis founder's algorithm)
Use 5 independent Redis masters; require lock acquisition on a **majority** within a bounded time. Aims to be safer than single-Redis locking.

**Controversy:** Martin Kleppmann famously argued Redlock is **unsafe under clock drift and process pauses**; Salvatore Sanfilippo (Redis author) defended it. The pragmatic takeaway: **distributed locks for correctness require fencing tokens** — see below.

### ZooKeeper / etcd locks
Race to create an ephemeral sequential node. Lowest sequence number wins; others watch the predecessor. If the holder crashes, the ephemeral node is deleted automatically and the next contender takes over.

**Stronger guarantees** than Redis-based locks because:

- Built on Raft/ZAB consensus.
- Session liveness is monitored.
- Leader election integrates naturally.
### Database-based
- `SELECT ... FOR UPDATE` 
- Postgres advisory locks (`pg_try_advisory_lock` )
- Unique constraint on a "leader" row
Simple, safe, but couples coordination to the DB.

## 5.3 Fencing Tokens (Critical for Correctness)
Every lock acquisition gets a **monotonically increasing token**. The protected resource (database, file system) **rejects writes with stale tokens**.

```
Client A acquires lock → token=42, then GC pause
Client B acquires lock → token=43, writes with token 43
Client A wakes up → tries to write with token 42 → rejected
```
Without fencing, **no distributed lock is truly safe** against pauses, clock drift, or partitions.

## 5.4 Lock Anti-Patterns
- **Long-held locks** — magnify the impact of pauses; prefer short critical sections.
- **No TTL** — a crashed holder blocks everyone forever.
- **Lock without fencing** — assumes locks are perfect; they aren't.
- **Lock as a substitute for proper transactions** — often you really want a database transaction.
---

# Part 6: Split Brain Problem
## 6.1 What It Is
When a network partition splits the cluster into two (or more) groups, and **each group thinks the other is dead and continues operating independently**, you get **split brain**: two leaders, two diverging histories, conflicting writes.

```
Network partition!
┌─────────────┐         ┌─────────────┐
│ Node A (L)  │         │ Node B      │
│ Node C      │   X X   │ Node D      │
│             │         │ (elects new │
│ "I'm leader"│         │  leader B)  │
└─────────────┘         └─────────────┘
Both halves accept writes → divergence
```
When the partition heals:

- Conflicting writes must be reconciled.
- Sometimes data is **lost** in the merge (last-write-wins) — silently.
## 6.2 How to Prevent It
### Quorum-based decisions
A node only acts as leader if it can talk to a **majority**. The minority side knows it doesn't have a majority and **refuses to serve writes** (this is the price of CP).

- Raft, Paxos, etcd, ZooKeeper, Kafka all require majority quorum for leader election and commit.
- This is why **odd cluster sizes (3, 5, 7)** are standard — even sizes can split evenly with no majority.
### Fencing
A new leader's epoch/term/token must be higher than any previous leader's. Old leaders' writes are rejected by storage layers that check tokens. Solves the "stale leader still writing" race.

### Witnesses / Arbiters
A small extra node (no data) breaks ties in 2-DC deployments. Common in MongoDB and Patroni setups.

### STONITH ("Shoot The Other Node In The Head")
In HA pair systems (Pacemaker, Postgres HA), the surviving node literally power-cycles or fences the other to be sure it's not still writing.

## 6.3 Where Split Brain Bites Hardest
- Old MongoDB versions (before majority writes were the default).
- Naively configured Elasticsearch (`minimum_master_nodes`  mistakes — pre-7.x).
- DIY Redis HA (without Sentinel quorum or Redis Cluster).
- Two-data-center deployments without an arbiter (50/50 split has no winner).
**Rule:** if you don't run an odd number of voters with strict quorum, you have a split-brain risk.

---

# Part 7: Time and Ordering in Distributed Systems
The fundamental problem: **clocks on different machines disagree**, and there is no shared "now." NTP keeps them within tens of milliseconds usually, but never perfectly. So how do we know what happened first?

## 7.1 Lamport Timestamps
Leslie Lamport (1978). The first solution to **logical ordering** without synchronized clocks.

### Rules
- Each process has a counter `L`  initialized to 0.
- On a local event: `L = L + 1` .
- When sending a message: include `L` .
- On receiving a message with timestamp `Lm` : `L = max(L, Lm) + 1` .
### Property
If event `A` happened-before event `B` (causally), then `L(A) < L(B)`.

### Limitation
The **converse is not true**: `L(A) < L(B)` does **not** imply `A` happened before `B`. Lamport timestamps give a **total order** consistent with causality but **can't detect concurrency** — they can't tell whether A→B, B→A, or they were concurrent.

### Where it's used
- Total ordering of events in distributed logs.
- Tie-breaking with `(timestamp, processId)`  to get a unique total order.
- Foundation for many other algorithms.
## 7.2 Vector Clocks
Generalization of Lamport timestamps to **detect concurrency and causality precisely**.

### Rules
- Each of `N`  processes has a vector `V[0..N-1]` , initialized to all zeros.
- Process `i`  on local event: `V[i] += 1` .
- Sending a message: include the whole vector.
- Receiving a message with vector `Vm` : `V[k] = max(V[k], Vm[k])`  for all k, then `V[i] += 1` .
### Comparison
For two events with vectors `Va` and `Vb`:

- `Va < Vb`  (every component ≤, at least one <) → A happened before B.
- `Vb < Va`  → B happened before A.
- Neither → A and B are **concurrent** (no causal relationship).
### Where it's used
- **Amazon DynamoDB / Dynamo paper** — to detect concurrent writes for conflict resolution.
- **Riak, Voldemort** — same.
- **Distributed version control** (Git uses similar concepts).
- **CRDTs** — to track causality of operations.
### Cost
The vector grows with the number of participants. For huge dynamic clusters, **dotted version vectors** or **interval tree clocks** are used to keep size manageable.

## 7.3 Other Ordering Approaches
- **Hybrid Logical Clocks (HLC)** — combine physical time + logical counter; bounded skew, used by CockroachDB.
- **TrueTime (Google Spanner)** — atomic clocks + GPS in datacenters give a **bounded uncertainty interval**; Spanner waits out the uncertainty before committing, achieving global external consistency. Hardware-dependent.
---

# Part 8: Gossip Protocol
## 8.1 What It Is
A **decentralized communication style** where nodes periodically exchange state with a few random peers. Information spreads exponentially, like a rumor — hence "gossip" or "epidemic" protocols.

```
t=0  Node A learns "X"
t=1  A → B          (B knows X)
t=2  A → C, B → D   (4 know X)
t=3  ...            (8 know X, then 16, 32 ...)
```
Convergence is **O(log N)** rounds for `N` nodes.

## 8.2 Why It Matters
- **No single point of failure** — no central coordinator.
- **Highly scalable** — each node only contacts a few peers per round, regardless of cluster size.
- **Resilient** — survives heavy node churn and partitions.
- **Eventually consistent** — all nodes converge over time.
## 8.3 What It's Used For
- **Cluster membership** — who's alive? (SWIM protocol, used by Consul, Serf, HashiCorp tools).
- **Failure detection** — heartbeats and suspicion via gossip.
- **State dissemination** — Cassandra spreads schema, ring topology, and load info.
- **Configuration sync** — distribute small config changes.
- **CRDT propagation** — Riak, Redis Active-Active.
- **Blockchains** — Bitcoin, Ethereum gossip transactions and blocks.
## 8.4 Properties
- **Probabilistic** — convergence is statistical, not guaranteed in any one round.
- **Bounded bandwidth per node** regardless of cluster size.
- **Best for "small, frequently changing state"** like membership/health, not large data transfer.
---

# Part 9: CRDTs — Conflict-free Replicated Data Types
## 9.1 The Problem
In an AP system, multiple replicas accept concurrent writes. When they sync, conflicts arise. Traditional approaches:

- **Last-write-wins** — silently discards writes (data loss).
- **Manual reconciliation** — burdens the application.
CRDTs solve this by **defining data types whose merges are mathematically guaranteed to converge**, regardless of the order or duplication of updates.

## 9.2 Two Families
### State-based CRDTs (CvRDT)
Replicas exchange entire state. Merge function must be:

- **Commutative** — `merge(a, b) = merge(b, a)` 
- **Associative** — `merge(a, merge(b, c)) = merge(merge(a, b), c)` 
- **Idempotent** — `merge(a, a) = a` 
Equivalent to a **join-semilattice**.

### Operation-based CRDTs (CmRDT)
Replicas broadcast operations. Operations must be **commutative** (delivered in any order, exactly once).

## 9.3 Examples
| CRDT | Behavior |
| ----- | ----- |
| **G-Counter** | Grow-only counter; per-replica counters, sum on merge |
| **PN-Counter** | Increment + decrement counters |
| **G-Set** | Grow-only set; union on merge |
| **2P-Set** | Add + tombstone set (can remove) |
| **OR-Set** | Observed-Remove set; tracks add-IDs to handle add/remove races |
| **LWW-Element-Set** | Last-write-wins set with timestamps |
| **RGA / Treedoc / Yjs / Automerge** | Collaborative text/sequence types |
## 9.4 Real-World Uses
- **Redis Enterprise Active-Active** — CRDTs power multi-region writeable replicas.
- **Riak** — CRDTs for counters, sets, maps.
- **Apple Notes, iCloud sync** — CRDT-like models for offline sync.
- **Figma, Google Docs (kinda)** — collaborative editing (Figma uses custom CRDTs; Google Docs uses Operational Transformation, a sibling approach).
- **Yjs and Automerge** — JavaScript libraries for collaborative apps.
- **CouchDB, AntidoteDB** — multi-master replication.
## 9.5 Trade-offs
✅ **Strong eventual consistency without coordination** — replicas can write offline and merge later.
✅ Great for **collaborative editing, offline-first apps, multi-region writeable databases**.
❌ **Memory overhead** — tombstones, version vectors, IDs accumulate.
❌ **Limited semantics** — not every data type has a CRDT; arbitrary business rules may not fit.
❌ **Complex to design correctly.**

## 9.6 CRDT vs Consensus
|  | **Consensus (Raft/Paxos)** | **CRDT** |
| ----- | ----- | ----- |
| Consistency | Strong | Strong eventual |
| Coordination | Required (majority) | None |
| Latency | Higher (round trips) | Local-first |
| Availability under partition | Reduced (CP) | Full (AP) |
| Use cases | Leader election, locks, atomic counters | Collaborative state, offline sync, multi-region writes |
They're **complementary** — different points on the consistency-availability curve.

---

# Part 10: How These Pieces Fit Together
A real distributed database (e.g., **CockroachDB** or **Cassandra**) combines many of these:

```
Client
  │
  ▼
Coordinator node
  │
  ├──► Hash key to partition
  ├──► Find replicas (via gossip-distributed topology)
  ├──► Use Raft to commit write across majority of replicas
  ├──► Quorum read (or consistent read via Raft lease)
  ├──► HLC timestamps order events across nodes
  ├──► Leader of each Raft group handles writes
  ├──► Split-brain prevented by majority voting
  └──► Failure detected via gossip / heartbeats
```
Or take **etcd / Kubernetes**:

- etcd nodes form a **Raft** group → consensus for all cluster state.
- API server reads/writes through the Raft leader.
- A **distributed lock** (etcd lease) is used for **leader election** of controllers (scheduler, controller-manager).
- **Quorum** of etcd nodes required for any write → split-brain impossible.
Or a **collaborative text editor (Figma)**:

- Each client has a local replica (CRDT).
- Edits applied locally instantly.
- Operations broadcast to peers / server.
- **No consensus needed** — CRDT guarantees convergence.
- **Vector clocks / similar** track causality.
---

# Part 11: When Each Concept Matters
| Concern | Apply |
| ----- | ----- |
| Choosing a database | <p>**CAP / PACELC**</p><p> to understand its trade-offs</p> |
| Replicated state machine, leader election | **Raft / Paxos** |
| "Run this once across the cluster" | <p>**Distributed lock**</p><p> + </p><p>**fencing tokens**</p> |
| Ensuring data integrity under failure | <p>**Quorum**</p><p> writes</p> |
| Multi-DC or HA design | <p>Plan to </p><p>**avoid split brain**</p><p> (odd voters, fencing, witnesses)</p> |
| Ordering events without synced clocks | **Lamport / vector clocks** |
| Cluster membership, large dynamic clusters | **Gossip protocol** |
| Multi-master writeable replicas, offline-first | **CRDTs** |
---

# Part 12: Universal Truths
1. **Networks are unreliable** — partitions, drops, reorders, duplicates are normal, not exceptional.
2. **Clocks lie** — never trust wall-clock time for ordering across machines.
3. **Majority quorums are the foundation** — most safe-by-design systems boil down to "ask a majority."
4. **Strong consistency costs latency or availability** — there's no escape from PACELC.
5. **Eventual consistency works if your data type is designed for it** — that's what CRDTs formalize.
6. **Fencing tokens save you from yourself** — never trust a lock without one.
7. **Split brain is the worst outcome**, worse than downtime — design defensively against it.
8. **Understandability matters** — Raft beat Paxos in adoption because engineers could implement it correctly.
---

# TL;DR
- **CAP**: under partition, choose Consistency or Availability. **PACELC**: even without partition, choose Latency or Consistency.
- **Paxos** and **Raft** solve consensus; Raft is preferred today for being understandable. They power etcd, Consul, CockroachDB, Kafka KRaft, MongoDB, and more.
- **Quorum** (majority voting) is the safety mechanism behind consensus, replication, and split-brain prevention.
- **Leader election** simplifies many distributed problems by funneling decisions through one node; built on consensus.
- **Distributed locks** are mutexes across machines — use Redis for soft locks, ZooKeeper/etcd for safe ones, and **always use fencing tokens** for correctness.
- **Split brain** = the cluster splits and both halves act independently. Prevented by quorum, fencing, witnesses, and odd cluster sizes.
- **Lamport timestamps** give a total order consistent with causality. **Vector clocks** detect both causality and concurrency, at the cost of size.
- **Gossip protocols** efficiently spread small state (membership, health) across large dynamic clusters with no central coordinator.
- **CRDTs** are data types whose merges are guaranteed to converge — enabling strong-eventually-consistent multi-master systems and offline-first collaboration without coordination.
Together, these are the building blocks behind almost every real distributed system you'll work with.



<!--- Eraser file: https://app.eraser.io/workspace/npnEbn0SAGt4lNd2T22e --->