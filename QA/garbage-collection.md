# Garbage Collection in Java — Notes
---

## 1. What is Garbage Collection (GC)?

**Garbage Collection** is the process by which the **JVM automatically reclaims memory** occupied by objects that are no longer reachable or referenced by the program.

- Eliminates the need for manual memory management (unlike C/C++ `free()`).
- Prevents **memory leaks** and **dangling pointers**.
- Runs as a **daemon thread** in the background.

---

## 2. JVM Memory Structure

```
┌─────────────────────────────────────────┐
│              JVM Memory                 │
├─────────────────────────────────────────┤
│  Heap (GC managed)                      │
│   ├── Young Generation                  │
│   │     ├── Eden                        │
│   │     ├── Survivor S0                 │
│   │     └── Survivor S1                 │
│   └── Old (Tenured) Generation          │
├─────────────────────────────────────────┤
│  Metaspace (class metadata)             │
│  Stack (per-thread, local vars)         │
│  PC Register                            │
│  Native Method Stack                    │
└─────────────────────────────────────────┘
```

| Area | Stores | GC? |
|------|--------|-----|
| **Heap** | Objects, instance variables | ✅ Yes |
| **Stack** | Local vars, method calls | ❌ Auto-cleared on method return |
| **Metaspace** | Class metadata (replaces PermGen in Java 8+) | Limited |

---

## 3. Heap Generations

### 🔹 Young Generation
- New objects are allocated here (in **Eden**).
- GC here is called **Minor GC** — fast and frequent.
- Survivors get moved between **S0 ↔ S1** spaces.

### 🔹 Old Generation (Tenured)
- Long-lived objects promoted from Young Gen.
- GC here is called **Major GC / Full GC** — slower, less frequent.

### 🔹 Metaspace (Java 8+)
- Stores class metadata.
- Replaced **PermGen** (removed in Java 8).
- Grows dynamically in native memory.

---

## 4. Object Lifecycle

1. Object created → allocated in **Eden**.
2. Eden full → **Minor GC** triggered.
3. Surviving objects → moved to **Survivor space** (S0 or S1).
4. After surviving N cycles (`-XX:MaxTenuringThreshold`) → promoted to **Old Gen**.
5. Old Gen full → **Major GC** triggered.
6. Unreachable → memory reclaimed.

---

## 5. How GC Identifies Garbage

### Reachability Analysis
An object is **eligible for GC** if no live thread can reach it via references from **GC Roots**.

**GC Roots include:**
- Local variables in active stack frames
- Active threads
- Static fields
- JNI references

### Ways an Object Becomes Eligible
```java
// 1. Nullifying reference
Employee e = new Employee();
e = null;

// 2. Reassigning reference
e = new Employee();

// 3. Object inside a method (out of scope after return)

// 4. Island of Isolation (objects reference each other but no GC root references them)
```

---

## 6. GC Algorithms

| Algorithm | Description |
|-----------|-------------|
| **Mark and Sweep** | Marks reachable objects, sweeps unreachable ones. Causes fragmentation. |
| **Mark-Sweep-Compact** | Adds compaction to reduce fragmentation. |
| **Copying** | Copies live objects to a new region (used in Young Gen). |
| **Generational** | Splits heap by age; basis of all modern Java GCs. |

---

## 7. Types of Garbage Collectors in Java

| Collector | Flag | Best For |
|-----------|------|----------|
| **Serial GC** | `-XX:+UseSerialGC` | Single-threaded, small apps |
| **Parallel GC** (Throughput) | `-XX:+UseParallelGC` | Multi-threaded, batch jobs (default in Java 8) |
| **CMS** (Concurrent Mark Sweep) | `-XX:+UseConcMarkSweepGC` | Low pause (deprecated in Java 9, removed in 14) |
| **G1 GC** (Garbage First) | `-XX:+UseG1GC` | Default since Java 9; balanced throughput + low pause |
| **ZGC** | `-XX:+UseZGC` | Very large heaps, sub-millisecond pauses (Java 11+) |
| **Shenandoah** | `-XX:+UseShenandoahGC` | Low-pause concurrent GC (Red Hat) |

---

## 8. `finalize()` Method

```java
@Override
protected void finalize() throws Throwable {
    // Cleanup code before GC
}
```

- Called by GC before reclaiming the object.
- **Deprecated since Java 9** — unreliable, slow.
- Use **`try-with-resources`** or **`Cleaner` API** instead.

---

## 9. Requesting GC (Not Guaranteed!)

```java
System.gc();         // Suggests JVM to run GC
Runtime.getRuntime().gc();
```

⚠️ JVM may **ignore** these calls. Never rely on them for correctness.

---

## 10. Types of References (java.lang.ref)

| Reference | GC Behavior |
|-----------|-------------|
| **Strong** | Default; never GC'd while reachable |
| **Soft** (`SoftReference`) | GC'd only when memory is low (good for caches) |
| **Weak** (`WeakReference`) | GC'd in next cycle (used in `WeakHashMap`) |
| **Phantom** (`PhantomReference`) | Used for pre-mortem cleanup with `ReferenceQueue` |

---

## 11. Important JVM Tuning Flags

```bash
-Xms512m              # Initial heap size
-Xmx2g                # Max heap size
-Xmn256m              # Young generation size
-XX:MetaspaceSize=128m
-XX:+PrintGCDetails
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
```

---

## 12. Advantages & Disadvantages

### ✅ Advantages
- Automatic memory management
- Prevents memory leaks & dangling pointers
- Increases developer productivity

### ❌ Disadvantages
- CPU overhead
- **Stop-the-world** pauses (app freezes briefly)
- Non-deterministic — can't predict exactly when GC runs

---

## 13. Best Practices

1. **Nullify** references no longer needed (for large objects).
2. Prefer **local variables** over instance/static.
3. Use **`try-with-resources`** to close resources.
4. Avoid **`System.gc()`** in production code.
5. Use **`StringBuilder`** instead of String concatenation in loops.
6. Choose appropriate GC based on workload (latency vs throughput).
7. Monitor with **JVisualVM, JConsole, GC logs, Eclipse MAT**.

---

## Quick Recap

> **GC = Automatic reclaiming of unreachable heap objects** → handled by JVM via generational algorithms (Young → Old) using collectors like **Serial, Parallel, CMS, G1, ZGC, Shenandoah**.
