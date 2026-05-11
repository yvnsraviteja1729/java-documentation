<p><a target="_blank" href="https://app.eraser.io/workspace/STSaw9cKrpgqal1E7rWd" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>



## Part 1: Thread Safety
---

### 1.1 What is Thread Safety?
A piece of code (a class, method, or data structure) is **thread-safe** if it **behaves correctly when accessed by multiple threads concurrently**, regardless of:

- The scheduling or interleaving of those threads.
- Without the caller having to perform additional external synchronization.
**Correct behavior** means:

- The class continues to meet its specification (invariants, postconditions).
- No data corruption.
- No lost updates.
- No visibility issues (every thread sees a consistent state).
- No surprise exceptions due to concurrent modification.
---

### 1.2 Formal Definition (from _Java Concurrency in Practice_)
>  A class is **thread-safe** if it behaves correctly when accessed from multiple threads, regardless of the scheduling or interleaving of the execution of those threads by the runtime environment, and with no additional synchronization or other coordination on the part of the calling code. 

The crucial phrase is **"no additional synchronization on the part of the calling code."** If callers must wrap operations in `synchronized` blocks themselves to get correct behavior, the class is **not** thread-safe — it's only **thread-compatible**.

---

### 1.3 Why is Thread Safety Hard?
In a single-threaded program, statements execute in a predictable order. In a multi-threaded program:

1. Threads may **interleave** at any point — even mid-instruction.
2. CPUs **cache** values in registers and L1/L2 caches → one thread may not see another's writes.
3. Compilers and CPUs **reorder** instructions for performance.
4. Operations that look atomic (`count++` , `x = y` ) often **aren't**.
5. Object construction can be **incompletely visible** to other threads.
Consider this seemingly innocent code:

```java
class Counter {
    private int count;
    public void increment() { count++; }
    public int get()        { return count; }
}
```
It looks fine, but it's **not thread-safe**:

- `count++`  is **read → modify → write** (3 separate operations).
- Two threads can read the same value, both increment, both write — one update is lost.
- Even `get()`  may return a stale value cached in another thread's CPU registers.
---

### 1.4 Three Pillars of Thread Safety
True thread safety requires **all three** of these:

| Pillar | Meaning | Mechanisms |
| ----- | ----- | ----- |
| **Atomicity** | Operations complete as an indivisible unit | `synchronized`, `Lock`, atomics |
| **Visibility** | Changes by one thread are seen by others | `volatile`, `synchronized`, atomics |
| **Ordering** | Instructions appear in expected order | `volatile`, `synchronized`, `Atomic*`  |
Missing any one of these leads to bugs.

---

### 1.5 Levels of Thread Safety
Brian Goetz (in _JCIP_) classifies code into these categories:

| Level | Description | Example |
| ----- | ----- | ----- |
| **Immutable** | State never changes after construction → always safe | `String`, `Integer`, `LocalDate`  |
| **Thread-safe** | Manages internal synchronization; safe with no caller effort | `ConcurrentHashMap`, `AtomicInteger`  |
| **Conditionally thread-safe** | <p>Individual operations safe, but </p><p>**sequences**</p><p> need external sync</p> | `Collections.synchronizedList` (iteration) |
| **Thread-compatible** | Not thread-safe by itself; safe if caller adds sync | `ArrayList`, `HashMap`  |
| **Thread-hostile** | Cannot be made safe even with external sync | Code mutating shared static state unsafely |
---

### 1.6 What Needs Protection?
You must protect **mutable shared state** — variables that:

1. Are **shared** between threads, AND
2. Are **mutable** (can change after construction).
**Inherently safe (no protection needed):**

- **Local variables** — live on each thread's own stack.
- **Method parameters of primitives or immutable types** — same reason.
- **Immutable objects** — fields are `final` , no setters, no internal mutation.
- **Stateless objects** — no instance fields at all.
- `**ThreadLocal**` ** variables** — each thread has its own copy.
**Needs protection:**

- Instance/static fields read or written by multiple threads.
- Mutable collections shared between threads.
- Even _reading_ without writing (still need visibility guarantees).
---

### 1.7 Six Strategies for Achieving Thread Safety
#### Strategy 1: Statelessness
A class with **no fields** (or only immutable ones) is automatically thread-safe.

```java
public class StatelessAdder {
    public int add(int a, int b) {
        return a + b;     // only local variables → thread-safe
    }
}
```
✅ Best when possible — many service/utility classes can be stateless.

#### Strategy 2: Immutability
An object whose state cannot change after construction is thread-safe forever.

**Rules for immutability:**

1. All fields are `final` .
2. The class is `final`  (or methods that mutate are absent).
3. No setters; no method changes state.
4. If holding references to mutable objects, **defensively copy** them on the way in and out.
5. Don't let `this`  escape during construction.
```java
public final class Point {
    private final int x, y;
    public Point(int x, int y) { this.x = x; this.y = y; }
    public int getX() { return x; }
    public int getY() { return y; }
}
```
Examples in the JDK: `String`, `Integer`, `LocalDateTime`, `UUID`, `BigInteger`.

#### Strategy 3: Synchronization (Mutual Exclusion)
Ensure only one thread accesses the critical section at a time.

```java
public class SynchronizedCounter {
    private int count;
    public synchronized void increment() { count++; }
    public synchronized int get()        { return count; }
}
```
`synchronized` provides two guarantees:

- **Mutual exclusion** → only one thread inside at a time.
- **Visibility** → changes by one thread become visible to the next thread acquiring the lock (happens-before).
#### Strategy 4: Atomic Variables
Use lock-free, hardware-supported atomic operations.

```java
import java.util.concurrent.atomic.AtomicInteger;

public class AtomicCounter {
    private final AtomicInteger count = new AtomicInteger();
    public void increment() { count.incrementAndGet(); }
    public int get()        { return count.get(); }
}
```
Best for **counters and flags**. Often faster than `synchronized` under contention. Backed by CAS (compare-and-swap).

#### Strategy 5: Concurrent Collections
Use purpose-built thread-safe collections.

```java
Map<String, Integer> counts = new ConcurrentHashMap<>();
counts.merge("apple", 1, Integer::sum);
```
These are far more scalable than `Collections.synchronizedMap(...)` because they use **fine-grained locking** or **lock-free algorithms** internally.

#### Strategy 6: Thread Confinement
Keep state accessible to only **one thread**. If only one thread can touch it, no sync is needed.

Three forms:

- **Stack confinement** — variables stay local to the method.
- `**ThreadLocal**`  — each thread has its own copy.
- **Single-threaded executor** — one task at a time.
```java
public class RequestContext {
    private static final ThreadLocal<String> userId = new ThreadLocal<>();

    public static void setUser(String id) { userId.set(id); }
    public static String getUser()        { return userId.get(); }
    public static void clear()            { userId.remove(); }  // ALWAYS clear!
}
```
⚠️ In thread pools, **always call **`**remove()**` to avoid leaks across reused threads.

---

### 1.8 Safe Publication
You can have a perfectly immutable object yet **still see broken state** if it's published unsafely. Example:

```java
// Thread A:
sharedRef = new Holder(42);

// Thread B:
sharedRef.checkInvariant();   // May see Holder with default int=0!
```
This happens because the JVM may reorder the constructor and reference write.

**Safe publication idioms:**

- Initializing through a `static`  initializer.
- Storing into a `volatile`  field or `AtomicReference` .
- Storing into a `final`  field of a properly constructed object.
- Storing into a field protected by a lock.
>  Rule: **All shared mutable state must be safely published.** 

---

### 1.9 Compound Actions — A Common Pitfall
Even if individual operations are atomic, **a sequence** of them may not be:

```java
if (!map.containsKey(key)) {     // ← atomic
    map.put(key, value);          // ← atomic
}                                  // but TOGETHER, not atomic
```
Between the two calls, another thread can interleave. Use atomic compound methods instead:

```java
map.putIfAbsent(key, value);          // single atomic operation
map.computeIfAbsent(key, k -> ...);   // even better
```
This is the **check-then-act** anti-pattern — one of the most common sources of concurrency bugs.

---

### 1.10 Visibility — Not Just Mutual Exclusion
A subtle aspect of thread safety: even if no two threads modify a variable simultaneously, one thread may **not see** another's write.

```java
class Worker {
    private boolean stop = false;          // NOT volatile

    public void run() {
        while (!stop) { /* work */ }       // may loop forever!
    }

    public void stop() { stop = true; }
}
```
The JIT may cache `stop` in a register inside the loop, so the writer thread's update is never seen.

**Fixes:**

- Declare `stop`  as `volatile` .
- Or use a synchronized accessor.
- Or use `AtomicBoolean` .
So thread safety = **atomicity + visibility + ordering**.

---

### 1.11 Documenting Thread Safety
A class's thread-safety contract should be **documented**. Java conventions:

- `@ThreadSafe`  — safe to use concurrently.
- `@Immutable`  — immutable (implies thread-safe).
- `@NotThreadSafe`  — not safe; caller must coordinate.
- `@GuardedBy("lock")`  — field/method protected by the named lock.
```java
@ThreadSafe
public class TaskQueue {
    @GuardedBy("this")
    private final List<Task> tasks = new ArrayList<>();
}
```
These annotations come from `net.jcip.annotations` or similar libraries.

---

### 1.12 Common Thread-Safe Building Blocks in the JDK
| Type | What it is |
| ----- | ----- |
| `String`, `Integer`, `Long` (wrappers), `BigInteger`, `LocalDate`  | Immutable |
| `AtomicInteger`, `AtomicLong`, `AtomicReference`, `LongAdder`  | Atomic primitives |
| `ConcurrentHashMap`, `ConcurrentLinkedQueue`, `CopyOnWriteArrayList`  | Concurrent collections |
| `BlockingQueue` impls (`ArrayBlockingQueue`, `LinkedBlockingQueue`) | Producer-consumer |
| `ReentrantLock`, `ReadWriteLock`, `StampedLock`  | Explicit locks |
| `CountDownLatch`, `CyclicBarrier`, `Semaphore`, `Phaser`  | Coordination |
| `CompletableFuture`  | Async composition |
| `ThreadLocalRandom`  | Per-thread RNG (avoids contention) |
---

### 1.13 Thread Safety Quick Checklist
✅ Identify all mutable shared state.
✅ Decide whether each piece needs protection.
✅ Choose a strategy: immutability, sync, atomic, concurrent collection, confinement.
✅ Watch out for compound actions.
✅ Ensure both atomicity **and** visibility **and** ordering.
✅ Verify safe publication (no escape of `this` from constructor).
✅ Document your choices with annotations.

---

## Part 2: Race Conditions
---

### 2.1 What is a Race Condition?
A **race condition** is a defect where the **correctness** of a program depends on the **relative timing or interleaving** of threads. In other words, the outcome "races" between possible execution orders, and at least one order is wrong.

The program may work correctly **most of the time** and fail rarely — making race conditions notoriously hard to detect and reproduce.

---

### 2.2 Formal Characterization
A race condition occurs when:

1. Two or more threads access the same data.
2. At least one of them performs a write.
3. The accesses are not properly synchronized.
4. The result depends on the order of operations.
>  **Note:** A race condition is a **logical bug**, while a **data race** is a **memory-model violation**. They often overlap but are not identical (more on this below). 

---

### 2.3 The Classic Example: Lost Update
```java
public class Counter {
    private int count = 0;

    public void increment() {
        count++;            // ← race condition here
    }

    public int get() { return count; }

    public static void main(String[] args) throws Exception {
        Counter c = new Counter();
        Runnable task = () -> {
            for (int i = 0; i < 100_000; i++) c.increment();
        };
        Thread t1 = new Thread(task);
        Thread t2 = new Thread(task);
        t1.start(); t2.start();
        t1.join();  t2.join();

        System.out.println(c.get());  // Expected 200_000 — often less!
    }
}
```
Why does this fail? `count++` is not atomic. It's actually three bytecode instructions:

```
1. getfield    count   (load into stack/register)
2. iconst_1; iadd      (add 1)
3. putfield    count   (store back)
```
Possible interleaving:

| Step | Thread A | Thread B | count |
| ----- | ----- | ----- | ----- |
| 1 | read count = 10 |  | 10 |
| 2 |  | read count = 10 | 10 |
| 3 | add 1 → 11 |  | 10 |
| 4 |  | add 1 → 11 | 10 |
| 5 | write 11 |  | 11 |
| 6 |  | write 11 | 11 |
Both threads incremented but the value went from 10 → 11, not 12. **An update was lost.**

---

### 2.4 Types of Race Conditions
There are several distinct kinds. Knowing them helps you spot bugs faster.

#### a) Read-Modify-Write (Lost Update)
Two threads read, modify, and write the same value, overwriting each other's changes.
Example: counter increment, `balance += amount`.

#### b) Check-Then-Act
Make a decision based on an observation, then act — but the observation may be stale by the time you act.

```java
if (!map.containsKey(k)) {   // check
    map.put(k, v);            // act
}
```
Or the famous broken singleton:

```java
if (instance == null) {       // check
    instance = new Singleton(); // act — TWO threads can both pass the check
}
```
#### c) Read-Read / Stale Read (Visibility Race)
One thread sees an out-of-date value because there's no happens-before relationship.

```java
// Thread A: data = 42; ready = true;
// Thread B: if (ready) use(data);  // may see ready=true but data=0
```
#### d) Compound Action on Collections
Multiple operations on a thread-safe collection that **individually** are atomic but **together** are not.

```java
List<String> list = Collections.synchronizedList(new ArrayList<>());
if (!list.contains(x)) list.add(x);   // not atomic
```
#### e) TOCTOU (Time-Of-Check / Time-Of-Use)
Same as check-then-act but more general — common in security contexts (file permissions, authentication, authorization).

#### f) Iteration Race
Iterating over a collection while another thread modifies it → `ConcurrentModificationException` or silent corruption.

#### g) Initialization Race
Object accessed before its constructor finishes (e.g., `this` escapes during construction).

---

### 2.5 Race Condition vs Data Race
People often use the terms interchangeably, but they're **subtly different**:

| Concept | Definition |
| ----- | ----- |
| **Data race** | <p>A </p><p>_memory-model_</p><p> violation: two threads access the same variable without proper synchronization, at least one writes. Result: undefined behavior, no happens-before.</p> |
| **Race condition** | <p>A </p><p>_correctness_</p><p> bug: program output depends on timing. May exist even without a data race (e.g., on thread-safe primitives used in compound actions).</p> |
Example: `check-then-act` on a `ConcurrentHashMap` is a **race condition** but **not** a data race — each individual call is properly synchronized.

>  Every data race is essentially a race condition, but not every race condition is a data race. 

---

### 2.6 Why Race Conditions Are So Dangerous
1. **Non-deterministic** — may work in dev, fail randomly in production.
2. **Reproduction is hard** — depend on CPU load, core count, scheduler decisions.
3. **Heisenbugs** — they disappear when you add logging or attach a debugger (timing changes).
4. **Silent corruption** — data may go wrong without any exception.
5. **Inconsistent state** — can cascade into more failures hours later.
6. **Security risk** — can be exploited (e.g., bypass authorization checks).
---

### 2.7 Real-World Examples
| Scenario | Race condition |
| ----- | ----- |
| Banking transfer | Two withdrawals at the same time both pass the balance check |
| E-commerce inventory | Two buyers see "1 left in stock" and both purchase it |
| ID generator | Two threads return the same "unique" ID |
| Caching | Two threads compute the same expensive value because both miss the cache |
| Login attempts | Brute-force lockout fails because the counter has a race condition |
| File creation | Two processes both check "file doesn't exist" then create it |
| Distributed locks | Two services both grab the lock due to clock skew |
---

### 2.8 How to Fix Race Conditions
The general principle: **make the conflicting access atomic with respect to other threads**, and **ensure proper happens-before** for visibility.

#### Fix 1: Synchronization
```java
public class Counter {
    private int count;
    public synchronized void increment() { count++; }
    public synchronized int get()        { return count; }
}
```
#### Fix 2: Atomic Variables
```java
private final AtomicInteger count = new AtomicInteger();
count.incrementAndGet();
```
Faster, lock-free, ideal for counters and flags.

#### Fix 3: Use Atomic Compound Methods
Replace check-then-act with built-in atomic operations:

| Instead of… | Use… |
| ----- | ----- |
| `if(!map.containsKey(k)) map.put(k,v);`  | `map.putIfAbsent(k,v);`  |
| `if(!map.containsKey(k)) map.put(k, expensive());`  | `map.computeIfAbsent(k, …);`  |
| `if(map.get(k).equals(v)) map.remove(k);`  | `map.remove(k, v);`  |
| `int old=v.get(); v.set(old+1);`  | `v.incrementAndGet();`  |
| `if(a == expected) a = newVal;`  | `atomicRef.compareAndSet(expected, newVal);`  |
#### Fix 4: Concurrent Collections
Replace `HashMap` + manual sync with `ConcurrentHashMap`, `CopyOnWriteArrayList`, etc.

#### Fix 5: Immutability
If state never changes, no race can occur. Recreate objects rather than mutate.

#### Fix 6: Thread Confinement
If only one thread touches a variable, no race. Examples: per-request `ThreadLocal`, single-threaded executor.

#### Fix 7: Higher-Level Coordinators
Use `BlockingQueue`, `Semaphore`, `CountDownLatch`, `Phaser`, `CompletableFuture` to structure cooperation instead of raw shared state.

#### Fix 8: Optimistic Concurrency (CAS / Versioning)
In databases and `StampedLock`, use a version/stamp to detect concurrent modification and retry.

---

### 2.9 Fixed Counter Example
```java
import java.util.concurrent.atomic.AtomicInteger;

public class ThreadSafeCounter {
    private final AtomicInteger count = new AtomicInteger();

    public void increment() { count.incrementAndGet(); }
    public int  get()       { return count.get(); }

    public static void main(String[] args) throws Exception {
        ThreadSafeCounter c = new ThreadSafeCounter();
        Runnable task = () -> {
            for (int i = 0; i < 100_000; i++) c.increment();
        };
        Thread t1 = new Thread(task), t2 = new Thread(task);
        t1.start(); t2.start();
        t1.join();  t2.join();
        System.out.println(c.get()); // always 200_000
    }
}
```
---

### 2.10 Detecting Race Conditions
Race conditions are hard to find. Use multiple tools and techniques.

| Approach | Description |
| ----- | ----- |
| **Code review** | Look for shared mutable state without synchronization |
| **Stress testing** | Run with many threads, many iterations, varying loads |
| <p>**jcstress**</p><p> (OpenJDK)</p> | The de-facto Java concurrency stress testing framework |
| `**-Xcomp**`**, **`**-XX:+StressLCM**`  | JVM stress flags |
| **FindBugs / SpotBugs** | Static analysis (`IS2_INCONSISTENT_SYNC`, `LI_LAZY_INIT_*`) |
| **Thread Sanitizer (TSan)** | Dynamic data-race detector (native) |
| **Java Flight Recorder** | Profiles thread states / contention |
| `**@GuardedBy**`** + annotation processors** | Enforce locking discipline |
| **Logging + invariant checks** | Detect inconsistent state in production |
| **Chaos engineering** | Inject artificial delays/sleeps to expose races |
---

### 2.11 Anti-Patterns That Cause Races
❌ Double-checked locking without `volatile`:

```java
if (instance == null) {
    synchronized(this) {
        if (instance == null) instance = new X();   // needs volatile
    }
}
```
❌ Synchronizing on `String` literals or boxed primitives:

```java
synchronized("LOCK") { ... }   // shared across the JVM!
synchronized(Integer.valueOf(1)) { ... }   // same — boxed cache
```
❌ Locking on a mutable field:

```java
synchronized(this.lock) { ... }  // if 'lock' reassigned, sync is broken
```
❌ Returning internal mutable state without copying:

```java
public List<Item> getItems() { return items; }   // caller can mutate
```
❌ Using `HashMap` from multiple threads — can cause **infinite loops** in Java 7 due to internal rehashing.

❌ Assuming `volatile` is enough for compound ops:

```java
private volatile int count;
count++;            // STILL a race!
```
❌ Letting `this` escape during construction (e.g., registering a listener inside the constructor before fields are set).

---

### 2.12 Race Conditions in Distributed Systems
The same logic extends beyond the JVM. In distributed systems you get races between **nodes/services**:

- Two requests both update the same DB row → use **optimistic locking** (version column) or **pessimistic locking** (`SELECT ... FOR UPDATE` ).
- Cache invalidation races → use distributed locks (Redis Redlock, Zookeeper).
- Double-spend in payments → idempotency keys for retries.
- Two leader elections → consensus protocols (Raft, Paxos).
The mental model is the same — only the synchronization primitives differ.

---

### 2.13 Quick Diagnostic Checklist
When you suspect a race condition:

1. ☐ Is there shared mutable state?
2. ☐ Is at least one thread writing it?
3. ☐ Is the write properly synchronized (lock / atomic / volatile + atomic op)?
4. ☐ Are there any **compound** operations (check-then-act, read-modify-write)?
5. ☐ Is visibility guaranteed (volatile, sync, happens-before)?
6. ☐ Are mutable objects safely published (no leaking `this`  from constructor)?
7. ☐ Do iterations over collections happen while other threads modify them?
8. ☐ Are any "thread-safe" collections being used in a non-atomic sequence?
---

## Summary
### Thread Safety
- A class is thread-safe when it works correctly under any thread interleaving **without** caller-side synchronization.
- Requires three pillars: **atomicity, visibility, ordering**.
- Achieve it through: **statelessness, immutability, synchronization, atomic variables, concurrent collections, or thread confinement**.
- Watch out for **compound actions** even on thread-safe types.
- Ensure **safe publication** of all shared mutable state.
### Race Conditions
- Race conditions arise from unsynchronized access to mutable shared state, especially in **read-modify-write** and **check-then-act** patterns.
- They are **non-deterministic, intermittent, and hard to reproduce**.
- Fix them by making the critical operation **atomic** and ensuring **visibility**.
- Use the right tool: `synchronized` , atomics, concurrent collections, immutability, or thread confinement.
- Detect them with stress testing, jcstress, static analysis, and disciplined code review.




<!--- Eraser file: https://app.eraser.io/workspace/STSaw9cKrpgqal1E7rWd --->