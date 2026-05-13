<p><a target="_blank" href="https://app.eraser.io/workspace/OXkPVXnJq1c8YOvcvXz5" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>

A practical guide to the caching tools, patterns, and pitfalls that show up in real distributed systems.

---

# Part 1: Why Cache?
Caching trades **memory and complexity** for **lower latency and reduced load** on the backing store. It's one of the highest-leverage optimizations in distributed systems — but also one of the **easiest to get wrong**.

>  "There are only two hard things in computer science: cache invalidation and naming things." — Phil Karlton 

A cache is useful when:

- Reads dominate writes.
- The same data is requested frequently.
- The backing store is slow, expensive, or rate-limited.
- Some staleness is acceptable.
It's the wrong tool when:

- Data is unique per request (no reuse).
- Strong consistency is non-negotiable for every read.
- The cost of stale data is high (financial balances, etc.).
---

# Part 2: Redis
## 2.1 What It Is
**Redis** ("Remote Dictionary Server") is an **in-memory, single-threaded, data-structure server**. It's the most widely used cache in production today, but it's much more than a cache: it's a database, message broker, stream platform, and coordination tool.

- Written in C, **single-threaded** for command execution (sub-millisecond latency, no lock contention).
- Networked, accessed over TCP via a simple text protocol (RESP).
- Supports **persistence** (RDB snapshots, AOF append-only file).
- **Replication** (primary–replica) and **Redis Cluster** for sharding.
- **Lua scripting** for atomic multi-key operations.
- **Streams**, **Pub/Sub**, **Geo**, **HyperLogLog**, and more.
## 2.2 Data Structures
This is what sets Redis apart from a simple key-value cache.

| Type | Use case |
| ----- | ----- |
| **String** | Cache values, counters, sessions |
| **Hash** | Object fields (user profile fields), partial updates |
| **List** | Queues, recent activity feeds |
| **Set** | Unique membership (online users), tag intersections |
| **Sorted Set** | Leaderboards, time-series, rate limiters |
| **Bitmap / Bitfield** | Feature flags, presence tracking, A/B groups |
| **HyperLogLog** | Approximate unique counts (cardinality) at fixed memory cost |
| **Stream** | Log-style append, consumer groups (Kafka-lite) |
| **Geo** | Lat/long search ("nearest restaurants") |
## 2.3 Common Caching Commands
```bash
SET user:42 "{...json...}" EX 300        # set with 5 min TTL
GET user:42
DEL user:42
INCR page_views:home
HSET user:42 name "Alice" age 30         # hash field set
HGET user:42 name
EXPIRE user:42 600                       # set/refresh TTL
TTL user:42                              # how long until expiry
SETNX lock:job:7 "owner1"                # set only if absent (lock primitive)
```
## 2.4 Persistence
Redis is in-memory, but can persist for durability:

- **RDB** — periodic point-in-time snapshots. Fast restart, possible data loss between snapshots.
- **AOF** (Append-Only File) — every write logged; rewritten/compacted periodically. Stronger durability, slightly slower.
- Often run with both enabled.
## 2.5 Scaling Redis
- **Replication** — primary handles writes; replicas serve reads / take over on failover (Redis Sentinel manages it).
- **Redis Cluster** — sharding across N primaries with hash slots (16384 slots assigned to nodes). Each shard can have replicas.
- **Memory eviction** when full: `noeviction` , `allkeys-lru` , `allkeys-lfu` , `volatile-ttl` , etc.
## 2.6 Why Redis Dominates
- Rich data structures → less app-side logic.
- Atomic operations + Lua scripting → safe concurrency primitives.
- Predictable, sub-millisecond latency.
- Battle-tested, ubiquitous, every cloud has a managed offering (ElastiCache, MemoryStore, Azure Cache).
- Doubles as a queue / coordination service if needed.
## 2.7 Watch-outs
- **Memory is finite** — set TTLs and an eviction policy; monitor `used_memory` .
- **Big keys hurt** (one giant hash, list, or set blocks the single thread). Avoid keys >100 KB; cap collection sizes.
- **Slow commands block everything** (`KEYS *` , `SMEMBERS hugeset` ). Use `SCAN`  instead.
- **Hot keys** can saturate a single shard. Shard hot keys (`product:1234:{0..15}` ) or use local cache in front.
- **Persistence ≠ durability guarantee** — can lose recent writes between fsyncs.
---

# Part 3: Memcached
## 3.1 What It Is
**Memcached** is a simpler, older, **multi-threaded in-memory key-value store** optimized for one job: fast caching of opaque blobs.

- Written in C, **multi-threaded** (scales well within a single node).
- Pure cache — **no persistence**, no replication.
- Values are opaque byte strings; no data structures.
- LRU eviction by default.
- Sharding is done **client-side** (consistent hashing).
- Tiny memory footprint per entry; very high throughput.
## 3.2 Memcached vs Redis
|  | **Redis** | **Memcached** |
| ----- | ----- | ----- |
| Threads | Single-threaded | Multi-threaded |
| Data structures | Many (lists, sets, hashes, streams, …) | Strings only |
| Persistence | RDB + AOF | None |
| Replication | Yes (Sentinel/Cluster) | No |
| Pub/Sub | Yes | No |
| Atomic scripting | Lua | Limited (CAS, INCR) |
| Eviction | Configurable policies | LRU only |
| Sharding | Redis Cluster | Client-side hashing |
| Use cases | Cache + DB + queue + coordination | Pure cache |
## 3.3 When to Use Memcached
- You need **a simple, fast, multi-threaded cache** for opaque values.
- You don't need data structures, persistence, or replication.
- Memory efficiency for **many small entries** matters (Memcached has slightly less overhead per item).
- You want **horizontal scaling within one node** (multi-threaded uses all cores natively).
In practice, **Redis has won** for most use cases because it's nearly as fast and far more capable. Memcached lives on in environments that already use it (large-scale web caching, session caches at companies like Facebook, Wikipedia).

---

# Part 4: Distributed Cache
## 4.1 What It Is
A cache that spans **multiple nodes** so it can hold more data than fits on one machine and survive node failures. Both Redis Cluster and Memcached (with client-side sharding) qualify.

```
┌──────── Application servers ────────┐
│  app1     app2     app3     app4    │
└────┬───────┬────────┬───────┬───────┘
     │       │        │       │
     ▼       ▼        ▼       ▼
┌────────────── Cache cluster ───────────┐
│ Shard 1   Shard 2   Shard 3   Shard 4 │
│ (k:0-3)   (k:4-7)   (k:8-B)   (k:C-F) │
└────────────────────────────────────────┘
```
## 4.2 Sharding (Partitioning)
Data is split across nodes by hashing the key:

- **Modulo hashing** (`hash(key) % N` ) — simple but disastrous when N changes (most keys move).
- **Consistent hashing** — keys move minimally when nodes join/leave (Memcached clients, DynamoDB, Cassandra, Riak).
- **Hash slots** — Redis Cluster's variant: 16384 fixed slots assigned to nodes; reassigning slots moves only the affected keys.
## 4.3 Replication
For availability and read scaling:

- **Primary–replica** (Redis): writes go to primary, replicas serve reads or take over on failure.
- **Multi-primary** (less common; complex conflict resolution).
## 4.4 Consistency Considerations
A distributed cache is **eventually consistent** with the source of truth (the database). Be aware of:

- **Replication lag** — read-your-write issues if you write to primary but read from replica.
- **Cache-DB skew** — covered in invalidation below.
- **Cross-shard atomicity** — none. A multi-key transaction across shards isn't possible without extra protocols.
## 4.5 Failure Modes
- **Single shard down** → its keyspace is unavailable. Either fail open (read from DB) or fail closed (return error).
- **Network partition** → some clients see stale data.
- **Cold cache** after restart → thundering herd against the DB. Mitigations: warm-up, request coalescing, stampede prevention (later).
---

# Part 5: Cache Invalidation
The hardest problem. There are several common strategies, each with tradeoffs.

## 5.1 TTL (Time-To-Live)
Every entry has an expiry. Simple, defensive, automatic.

```
SET user:42 "..." EX 300   # expires in 5 minutes
```
- ✅ No coordination needed.
- ❌ Up to TTL of staleness — sometimes unacceptable.
- ❌ Mass expiry → stampede (mitigate with **TTL jitter**: random TTL ±10%).
## 5.2 Explicit Invalidation on Write
When the source of truth changes, **delete the cache entry** (preferred) or **update it**.

```python
update_user_in_db(user)
cache.delete(f"user:{user.id}")   # next read will refresh
```
- ✅ Tight consistency.
- ❌ Doesn't help if the DB write succeeds and the cache delete fails (race) — combine with TTL as a safety net.
- ❌ Multiple writers must all remember to invalidate — use a write-through pattern instead.
## 5.3 Write-Through / Write-Behind
The cache layer **owns the invalidation logic** because all writes go through it. (Detailed below.)

## 5.4 Versioned Keys
Embed a version in the key; old entries are simply orphaned and aged out.

```
product:42:v17     ← current
product:42:v16     ← orphan, expires by TTL
```
Bumping the version "invalidates" everything atomically. Useful for cache-keyed collections that must change wholesale.

## 5.5 Event-Driven Invalidation
Publish change events on a bus (Kafka, Redis Pub/Sub, Debezium CDC). Cache nodes subscribe and invalidate locally.

```
DB → CDC → Kafka → cache invalidator → DEL keys
```
- ✅ Decoupled, scalable, works across services and regions.
- ✅ Handles "the writer doesn't know who's caching."
- ❌ Pipeline complexity; some lag.
## 5.6 The Race-Condition Trap
Common bug: **update DB then update cache**.

```
T1: write DB  v=1
T2: write DB  v=2
T2: write cache v=2
T1: write cache v=1   ← old value wins! cache now stale
```
Mitigations:

- **Delete (don't update) on write** — next read repopulates from DB authoritatively.
- **Versioning / CAS** — check version on cache write.
- **Single-writer per key** — possible with a queue.
- **TTL safety net** — bound the staleness even if you screw up.
---

# Part 6: Near Cache (Local / In-Process Cache)
## 6.1 What It Is
A cache that lives **inside the application process** — same JVM/Node/Python heap. Sub-microsecond access, no network hop. Often used in front of a remote distributed cache.

```
┌─────────────────────────────┐
│  Application process        │
│  ┌───────────────────────┐  │
│  │  Near Cache           │  │  ← L1 (in-process, very fast)
│  │  (Caffeine, Guava)    │  │
│  └─────────┬─────────────┘  │
└────────────┼────────────────┘
             │ miss
             ▼
┌────────────────────────┐
│ Distributed Cache      │  ← L2 (Redis, Memcached)
│ (Redis cluster)        │
└─────────┬──────────────┘
          │ miss
          ▼
    ┌──────────┐
    │ Database │  ← source of truth
    └──────────┘
```
This is a **multi-tier cache** (L1 in-process + L2 distributed + L3 DB). Frameworks like Hazelcast, Apache Ignite, and Infinispan call it "near cache."

## 6.2 Pros
- **Fastest possible reads** (no serialization, no network).
- **Reduces load** on the distributed cache.
- **Survives short Redis outages** for reads.
## 6.3 Cons (and they're real)
- **Per-process state** → cache size × N processes = wasted memory.
- **Hardest invalidation problem** — every process holds its own copy. Strategies:
    - Short TTLs only (e.g., 5–30 seconds).
    - Pub/Sub invalidation: when data changes, broadcast a "drop key X" message to all processes.
    - **Redis 6+ Client-Side Caching** (`CLIENT TRACKING` ) — Redis tracks which clients cached which keys and notifies them on change. Excellent feature for near caches.

- Risk of **stale reads** longer than your distributed cache would have.
## 6.4 When to Use
- Extremely hot, mostly-read data (config, feature flags, reference data).
- Latency-critical paths where even a 1 ms Redis hop matters.
- When request volume to Redis is becoming a bottleneck.
### Popular libraries
- **Java**: Caffeine (the modern winner), Guava, Ehcache, Hazelcast (with near-cache), Infinispan.
- **Go**: Ristretto, BigCache.
- **Node**: lru-cache, Keyv.
- **Python**: cachetools, functools.lru_cache.
---

# Part 7: Cache Patterns — Read-Through, Write-Through, Cache-Aside, Write-Behind
These describe **who is responsible for moving data between cache and DB**.

## 7.1 Cache-Aside (Lazy Loading) — the most common
The application owns the logic. The cache is dumb.

```python
def get_user(id):
user = cache.get(f"user:{id}")
if user is None:
    user = db.get_user(id)         # miss → load
    cache.set(f"user:{id}", user, ttl=300)
return user
```
- ✅ Simple, only used data is cached.
- ❌ First request after miss is slow (cold-cache penalty).
- ❌ App must handle invalidation everywhere it writes.
## 7.2 Read-Through Cache
The cache itself knows how to load from the DB on a miss. The app just calls `cache.get(id)`.

```
app.get(id)  →  cache.get(id)  →  [miss] → loader.load(id) → DB
  ↓
cache it, return
```
- ✅ Centralizes load logic.
- ✅ Cleaner app code.
- ❌ Cache must have a configured "loader" function.
- ❌ First request still slow (mitigated by warm-up/prefetch).
Implemented by: **Caffeine **`**LoadingCache**`, **Hazelcast MapLoader**, **Spring **`**@Cacheable**`, **Guava **`**CacheLoader**`.

## 7.3 Write-Through Cache
Every write goes to the cache **and** the DB **synchronously, in order**.

```
app.set(key, val) → cache.set → DB.write → return
```
- ✅ Cache always consistent with DB on writes.
- ✅ Reads always hit a warm cache.
- ❌ Writes are slower (two systems involved).
- ❌ Caches data that might never be read (memory waste).
- Common in: enterprise caches (Hazelcast write-through), some ORMs.
## 7.4 Write-Behind (Write-Back) Cache
Writes go to the cache immediately; flushed to the DB **asynchronously** in batches.

```
app.set → cache.set → return
  │
  ▼ (background flush)
DB.batchWrite
```
- ✅ Very fast writes; great for write-heavy workloads.
- ✅ Batches reduce DB load.
- ❌ **Risk of data loss** if cache node dies before flushing.
- ❌ Read-after-write consistency between cache and DB is eventual.
- Use when throughput matters more than per-write durability (analytics, counters).
## 7.5 Choosing
| Pattern | Read latency | Write latency | Consistency | Risk of data loss | Use when |
| ----- | ----- | ----- | ----- | ----- | ----- |
| Cache-aside | Low (after warm) | Same as DB | App-managed | None | **Default for most apps** |
| Read-through | Low (after warm) | Same as DB | App-managed | None | Want centralized load logic |
| Write-through | Low | Higher | Strong cache↔DB | None | Read-heavy, need fresh cache |
| Write-behind | Low | Lowest | Eventual | Yes | Write-heavy, durability flexible |
---

# Part 8: Cache Stampede Prevention
## 8.1 What Is a Cache Stampede?
Also called **dogpile** or **thundering herd**. When a cached value expires (or a hot key is missing), **many concurrent requests miss the cache simultaneously**, all hammer the database to recompute the same value, and crush it.

```
Cache expires
             │
             ▼
1000 requests arrive at once
             │
             ▼
1000 cache misses
             │
             ▼
1000 DB queries for the same row
             │
             ▼
    💥 DB falls over
```
This is one of the most common production outages caused by caching done naively.

## 8.2 Mitigations
### 1. Request Coalescing / Single-Flight
Only **one** request actually computes the value; others wait for the result. Implement with an in-process lock keyed by the cache key.

```go
// pseudo-code
val, err := singleFlight.Do(key, func() (any, error) {
    return loadFromDB(key)
})
```
Libraries: Go's `singleflight`, Caffeine's `LoadingCache.get` (atomic), Guava `LoadingCache`.

### 2. Distributed Lock on Recompute
Across processes, use a Redis lock so only one node recomputes a hot key.

```
lock = redis.SET("lock:user:42", uuid, NX, EX=10)
if lock acquired:
    compute and set cache
    release lock
else:
    sleep + retry / serve stale
```
Use **Redlock** carefully or simple `SETNX` with TTL.

### 3. Probabilistic / Early Expiration (XFetch)
Recompute **before** the entry expires, with a probability that grows as expiry approaches. Smooths the recompute load.

```python
if random() < probability_based_on_remaining_ttl:
recompute_in_background()
```
Algorithm: **XFetch** (Vattani, Chierichetti, Lowenstein, 2015).

### 4. Stale-While-Revalidate (SWR)
Serve the **stale value** to clients while a single background task refreshes it. Clients never wait. Common in HTTP caching (`Cache-Control: stale-while-revalidate`) and in libraries like SWR (React).

### 5. TTL Jitter
Add randomness (±10–20%) to TTLs so a million keys don't all expire at the same instant.

```python
ttl = base_ttl + random.randint(-30, 30)
```
### 6. Pre-warming
After deploys / cache flushes, proactively populate hot keys before serving live traffic.

### 7. Negative Caching (Cache Misses)
If a query returns "not found," **cache that fact** with a short TTL so repeated lookups for nonexistent keys don't keep hitting the DB.

```python
cache.set(f"user:{id}", "__NULL__", ttl=60)
```
### 8. Circuit Breaker on the DB
If the DB is overloaded, fail fast (return cached/stale/default) rather than amplifying the storm.

### 9. Bulkheading
Cap concurrent DB connections per service so a stampede can't exhaust the connection pool.

### Combining strategies
A robust setup typically uses **TTL jitter + single-flight + stale-while-revalidate + negative caching + monitoring**. None of these alone is bulletproof; together they make stampedes a non-event.

---

# Part 9: Other Critical Pitfalls
### Hot Keys
A single famous key (a viral product, celebrity user) overwhelms one shard.

- Shard the key (`product:1234:{0..15}` , pick randomly).
- Add a near cache in front so most reads never hit Redis.
- Use a CDN for public, cacheable responses.
### Big Keys
A single 1 MB hash blocks Redis's single thread on every read/write.

- Cap sizes; split across multiple keys.
- Use `MEMORY USAGE`  / `redis-cli --bigkeys`  to find offenders.
### Cache Penetration
Attackers query nonexistent IDs (e.g., `user:-1`, `user:9999999999`). All miss the cache → all hit the DB.

- **Negative caching** (above).
- **Bloom filter** in front of the cache to reject "definitely doesn't exist" lookups.
### Cache Avalanche
A whole cache cluster goes down or restarts → every read is a miss → DB melts.

- Multi-tier cache (near cache absorbs some).
- Circuit breaker + fallback.
- Gradual restart with warm-up.
### Eviction Surprises
You configured `noeviction` and Redis is now refusing writes because it's full. Or you used `allkeys-lru` and it evicted critical session data because nothing had a TTL. **Match your eviction policy to your usage** and **monitor memory + evictions**.

---

# Part 10: When and What to Cache — Quick Heuristics
| Data | Cache? | TTL guidance |
| ----- | ----- | ----- |
| Reference data (countries, currencies) | ✅ Heavily, near cache | Hours |
| User profile / preferences | ✅ | Minutes |
| Auth tokens / sessions | ✅ Redis | Token lifetime |
| Catalog / product data | ✅ Multi-tier | Minutes |
| Search results | ✅ Per-query | 30s–5m |
| Real-time financial balance | ❌ Or with care | Seconds, with versioning |
| Per-user feed | Depends — short TTL or compute on read | Seconds |
| Computed aggregates / counters | ✅ With write-behind | Variable |
| Personalized + unique per request | ❌ | — |
---

# Part 11: Architecture Stack — How It All Fits
```
┌──── CDN (HTTP cache, edge) ────┐         ← public, geographic
│                                │
│     API gateway (cache hdrs)   │
│              │                 │
│              ▼                 │
│   ┌────────────────────────┐   │
│   │ App process            │   │
│   │  • Near cache (L1)     │   │  ← Caffeine, microsecond hits
│   └────────┬───────────────┘   │
│            │ miss              │
│            ▼                   │
│   ┌────────────────────────┐   │
│   │ Distributed cache (L2) │   │  ← Redis cluster, millisecond hits
│   └────────┬───────────────┘   │
│            │ miss              │
│            ▼                   │
│      ┌──────────┐              │
│      │ Database │              │  ← source of truth
│      └──────────┘              │
└────────────────────────────────┘
```
Invalidation flows top-down on write:

1. App writes to DB.
2. App invalidates Redis.
3. Pub/Sub or client-side tracking notifies all near caches.
4. CDN cache key rotates (versioned URL) or is purged.
---

# Part 12: TL;DR
- **Redis** — versatile in-memory data structure server; the default cache (and more) for modern systems.
- **Memcached** — simpler, multi-threaded, opaque blob cache; mostly displaced by Redis but still in use.
- **Distributed cache** — sharded + replicated cache across nodes; gives capacity and availability with eventual consistency.
- **Cache invalidation** — combine TTLs, explicit deletes on write, versioned keys, and event-driven invalidation. **Always have a TTL safety net.**
- **Near cache** — in-process L1 cache in front of Redis; fastest reads but invalidation is hardest (use Pub/Sub or Redis 6+ client-side caching).
- **Read-through** — cache loads from the DB on miss; centralizes load logic.
- **Write-through** — writes go to cache and DB together; keeps cache fresh, slower writes.
- **Write-behind** — async DB writes from cache; very fast, but risk of data loss.
- **Cache stampede prevention** — TTL jitter, single-flight/coalescing, distributed locks, stale-while-revalidate, negative caching, pre-warming. Combine them.
- Watch for **hot keys, big keys, penetration, avalanches, and eviction surprises** — these are where caching outages actually come from.
>  Caching is easy to add and hard to operate. Design for invalidation and stampedes from day one. 





<!--- Eraser file: https://app.eraser.io/workspace/OXkPVXnJq1c8YOvcvXz5 --->