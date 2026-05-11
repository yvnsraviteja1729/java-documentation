<p><a target="_blank" href="https://app.eraser.io/workspace/mfQnts0WVgDWoHrtsWoD" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>

1. Introduction to Concurrency
2. Process vs Thread
3. Thread Lifecycle
---

## 1. Introduction to Concurrency
### 1.1 What is Concurrency?
**Concurrency** is the ability of a program to make progress on **more than one task at the same time**. The tasks may not literally run in parallel on different CPU cores — they may simply be **interleaved** by the operating system / JVM scheduler so that, from an outside observer's perspective, they all "advance" together.

A simple way to think about it:

- **Sequential**: Do task A completely, then do task B.
- **Concurrent**: Switch back and forth between A and B so both progress.
- **Parallel**: Truly do A and B at the same instant on different CPU cores.
### 1.2 Concurrency vs Parallelism
| Concept | Definition | Example |
| ----- | ----- | ----- |
| **Concurrency** | Dealing with many tasks at once (logical) | One chef cooking 3 dishes by switching between them |
| **Parallelism** | Doing many tasks at once (physical) | Three chefs each cooking one dish |
>  Concurrency is about **structure**; parallelism is about **execution**.
A program can be **concurrent but not parallel** (single-core machine running many threads), or **parallel but not concurrent** (SIMD instructions), or **both** (multi-threaded program on a multi-core CPU). 

### 1.3 Why Do We Need Concurrency?
1. **CPU utilization** — Modern machines have many cores (8, 16, 64+). A single-threaded program wastes them.
2. **Responsiveness** — UI applications shouldn't freeze while doing background work (e.g., file download).
3. **Throughput** — A web server must handle thousands of clients simultaneously.
4. **Latency hiding** — While one thread is blocked on I/O (disk, network, DB), another thread can do useful work.
5. **Natural modeling** — Some problems are inherently concurrent: chat apps, simulations, games, message brokers.
### 1.4 Types of Concurrency
| Type | Description | Example |
| ----- | ----- | ----- |
| **CPU-bound** | Tasks limited by CPU speed | Image processing, encryption |
| **I/O-bound** | Tasks mostly waiting on I/O | DB calls, REST APIs, file reads |
| **Mixed** | Both CPU + I/O | Web servers, ETL pipelines |
This distinction matters for thread pool sizing (covered later).

### 1.5 How Java Supports Concurrency
Java has had concurrency built into the language since version 1.0. Over time it has matured significantly:

| Java Version | Feature |
| ----- | ----- |
| 1.0 | `Thread`, `Runnable`, `synchronized`, `wait/notify`, `volatile`  |
| 1.5 | `java.util.concurrent` (Executors, Locks, Atomics, ConcurrentCollections), JMM revised (JSR-133) |
| 1.7 | `ForkJoinPool`, `Phaser`  |
| 1.8 | `CompletableFuture`, parallel streams, `StampedLock`, `LongAdder`  |
| 9 | `Flow` API (Reactive Streams) |
| 21 | <p>**Virtual Threads**</p><p> (Project Loom), Structured Concurrency (preview)</p> |
| 25 | Structured Concurrency (stabilizing) |
### 1.6 Key Challenges of Concurrent Programming
Concurrency is powerful but **hard**. The same code can produce different outputs on different runs due to non-deterministic scheduling.

| Challenge | Description |
| ----- | ----- |
| **Race conditions** | Output depends on thread interleaving |
| **Deadlocks** | Two threads waiting forever on each other's locks |
| **Livelocks** | Threads keep reacting to each other, no progress |
| **Starvation** | A thread never gets CPU/lock |
| **Memory visibility** | One thread doesn't see another's writes (due to CPU caches) |
| **Instruction reordering** | Compiler/CPU reorders instructions for performance |
| **Debugging difficulty** | Bugs are non-deterministic and hard to reproduce |
| **Performance overhead** | Context switching, locking, cache coherence |
### 1.7 A Simple Concurrency Example
```java
public class HelloConcurrency {
    public static void main(String[] args) {
        Runnable task = () -> {
            for (int i = 0; i < 5; i++) {
                System.out.println(Thread.currentThread().getName() + " - " + i);
            }
        };

        Thread t1 = new Thread(task, "Worker-1");
        Thread t2 = new Thread(task, "Worker-2");

        t1.start();
        t2.start();
    }
}
```
Possible output (will differ on every run):

```
Worker-1 - 0
Worker-2 - 0
Worker-1 - 1
Worker-2 - 1
Worker-2 - 2
Worker-1 - 2
...
```
The interleaving is unpredictable — that's the essence of concurrency.

### 1.8 Mental Model
Think of concurrency in three layers:

1. **Tasks** — units of work (a `Runnable` , a `Callable` ).
2. **Execution mechanism** — how tasks run (`Thread` , `ExecutorService` , virtual thread).
3. **Coordination** — how tasks communicate safely (locks, atomics, queues, futures).
A good concurrent design picks the right abstraction at each layer rather than manually managing threads.

---

## 2. Process vs Thread
To understand threads, you must first understand processes — the OS-level container threads live in.

### 2.1 What is a Process?
A **process** is an instance of a program in execution. It has:

- Its **own address space** (code, heap, stack, data segments).
- **OS resources** (file descriptors, sockets, environment variables).
- **At least one thread** (the "main" thread).
- A **process ID (PID)**.
When you start `java MyApp`, the OS creates one process; the JVM runs inside it.

### 2.2 What is a Thread?
A **thread** is the smallest unit of execution **inside a process**. Threads within the same process:

- **Share** the process's heap, code, and open file descriptors.
- Have their **own stack**, program counter (PC), and registers.
- Are scheduled independently by the OS.
So if a process is a "house," threads are "people inside the house" — they share the kitchen and living room (heap) but each has their own bedroom (stack).

### 2.3 Detailed Comparison
| Aspect | Process | Thread |
| ----- | ----- | ----- |
| **Definition** | An independent program in execution | A unit of execution within a process |
| **Memory** | Separate (isolated) address space | Shared heap; each thread has its own stack |
| **Creation cost** | Heavy (OS allocates page tables, file descriptors, etc.) | Lightweight |
| **Context switch cost** | Expensive (MMU flush, TLB invalidate) | Cheaper (no address space switch) |
| **Communication** | Inter-Process Communication: pipes, sockets, shared memory, signals | Shared variables in heap (must be synchronized) |
| **Fault isolation** | Crash in one process doesn't affect others | Crash in one thread can crash the whole process (JVM) |
| **Security** | OS-enforced isolation | No isolation between threads |
| **Resources** | Owns file handles, sockets, etc. | Shares with other threads in the process |
| **Identifier** | PID | Thread ID (TID) within process |
| **Examples** | `chrome.exe`, `java.exe`  | Worker thread in a pool |
### 2.4 Memory Layout
```
┌────────────────────────────────────┐
│            PROCESS                 │
│  ┌───────────────────────────────┐ │
│  │   HEAP (shared by threads)    │ │
│  │   - objects, class metadata   │ │
│  └───────────────────────────────┘ │
│  ┌──────────┐  ┌──────────┐        │
│  │ Thread 1 │  │ Thread 2 │  ...   │
│  │  Stack   │  │  Stack   │        │
│  │  PC,Regs │  │  PC,Regs │        │
│  └──────────┘  └──────────┘        │
│  ┌───────────────────────────────┐ │
│  │   Code (text segment)         │ │
│  │   Open file descriptors       │ │
│  └───────────────────────────────┘ │
└────────────────────────────────────┘
```
Each thread has its own **call stack** where local variables and method frames live → that's why **local variables are thread-safe by default**.

### 2.5 In Java Specifically
When you run a Java program:

1. OS starts a JVM **process**.
2. JVM starts the **main thread** (which runs `main()` ).
3. JVM also starts several **daemon threads**:
    - GC threads
    - JIT compiler thread
    - Finalizer thread
    - Signal dispatcher
    - Reference handler

You can list all live threads:

```java
Thread.getAllStackTraces().keySet()
.forEach(t -> System.out.println(t.getName() + " (daemon=" + t.isDaemon() + ")"));
```
### 2.6 User Threads vs Daemon Threads
| User Thread | Daemon Thread |
| ----- | ----- |
| Keeps JVM alive | JVM exits when only daemons are left |
| Default | Must call `setDaemon(true)` before `start()`  |
| Example: main, app threads | Example: GC, finalizer |
```java
Thread t = new Thread(task);
t.setDaemon(true); // must be set before start()
t.start();
```
### 2.7 Why Threads Instead of Processes?
| Benefit | Explanation |
| ----- | ----- |
| **Cheaper** | Spawning a thread costs ~microseconds; a process costs ~milliseconds |
| **Fast communication** | Just read/write shared variables vs IPC |
| **Lower memory** | One process's heap shared, no duplication |
| **Faster context switch** | No address-space change |
But threads come with a trade-off: **no isolation** → a bug in one thread (e.g., infinite loop, OOM) can damage the entire process.

### 2.8 Multi-Process vs Multi-Threaded Architectures
| Approach | Examples | When to use |
| ----- | ----- | ----- |
| **Multi-process** | Chrome (tab per process), Postgres, nginx (worker processes) | When isolation/security matters; languages without good threads (older Python due to GIL) |
| **Multi-threaded** | Java app servers, databases (e.g., MySQL connections), most JVM apps | When you need shared memory and high throughput |
### 2.9 Quick Recap
- **Process** = isolated OS-level container with its own memory.
- **Thread** = lightweight execution unit inside a process; shares memory with siblings.
- Java applications are **multi-threaded by default** (main + JVM daemons).
- Shared memory makes threads fast but also makes synchronization a necessity.
---

## 3. Thread Lifecycle
A thread goes through well-defined states from creation to termination. Java models these using the `Thread.State` enum.

### 3.1 The Six Thread States
```java
public enum Thread.State {
    NEW,
    RUNNABLE,
    BLOCKED,
    WAITING,
    TIMED_WAITING,
    TERMINATED
}
```
### 3.2 State Diagram
```
┌──────┐
                │ NEW  │
                └──┬───┘
              start()│
                   ▼
              ┌──────────┐
              │ RUNNABLE │◄──────────────────────┐
              └────┬─────┘                       │
                   │                             │
   ┌───────────────┼───────────────┐             │
   │               │               │             │
waiting for     wait()/         sleep(t)/        │
   lock         join()          wait(t)          │
   │               │               │             │
   ▼               ▼               ▼             │
┌────────┐   ┌─────────┐    ┌──────────────┐     │
│BLOCKED │   │ WAITING │    │TIMED_WAITING │─────┘
└───┬────┘   └────┬────┘    └──────┬───────┘
    │             │                │
 lock acquired notify/notifyAll  time elapsed
    └─────────────┴────────────────┘
                  │
                  ▼
             run() returns
                  │
                  ▼
             ┌────────────┐
             │ TERMINATED │
             └────────────┘
```
### 3.3 State-by-State Explanation
#### a) NEW
- Thread object is created but `start()`  hasn't been called.
- No OS-level thread exists yet.
```java
Thread t = new Thread(() -> {});
System.out.println(t.getState()); // NEW
```
#### b) RUNNABLE
- After `start()`  is called.
- Thread is **eligible** to run — it's either:
    - **Currently running** on a CPU core, OR
    - **Ready to run** but waiting for CPU.

- Java does **not** distinguish these two — both are `RUNNABLE` .
```java
t.start();
System.out.println(t.getState()); // RUNNABLE
```
#### c) BLOCKED
- Thread is waiting to acquire a **monitor lock** (entering a `synchronized`  block/method that's owned by another thread).
- Cannot do anything else until the lock is free.
```java
synchronized (lock) {
    // If another thread holds 'lock', this thread is BLOCKED here.
}
```
#### d) WAITING
- Thread is **waiting indefinitely** for another thread to signal it.
- Entered via:
    - `Object.wait()`  (no timeout)
    - `Thread.join()`  (no timeout)
    - `LockSupport.park()` 

- Must be woken up by:
    - `Object.notify()`  / `notifyAll()` 
    - Target thread completes (`join` )
    - `LockSupport.unpark()` 

#### e) TIMED_WAITING
- Same as WAITING, but with a **timeout**.
- Entered via:
    - `Thread.sleep(ms)` 
    - `Object.wait(ms)` 
    - `Thread.join(ms)` 
    - `LockSupport.parkNanos(...)`  / `parkUntil(...)` 

- Automatically exits when time elapses or it's signaled.
#### f) TERMINATED
- `run()`  has finished (normally or via uncaught exception).
- A thread **cannot be restarted** — calling `start()`  again throws `IllegalThreadStateException` .
### 3.4 Quick Reference Table
| State | Entered When | Exits When |
| ----- | ----- | ----- |
| NEW | `new Thread()`  | `start()` is called |
| RUNNABLE | `start()` invoked / unblocked | Blocked / waits / completes |
| BLOCKED | Waiting for monitor lock | Lock acquired |
| WAITING | `wait()`, `join()`, `park()` (no timeout) | `notify`, `notifyAll`, `unpark`, target ends |
| TIMED_WAITING | `sleep`, `wait(t)`, `join(t)`, `parkNanos`  | Time expires or signaled |
| TERMINATED | `run()` returns or throws | (final state) |
### 3.5 Code Example Demonstrating All States
```java
public class ThreadStateDemo {

    private static final Object lock = new Object();

    public static void main(String[] args) throws InterruptedException {

        Thread worker = new Thread(() -> {
            try {
                Thread.sleep(500);                  // TIMED_WAITING
                synchronized (lock) {               // may be BLOCKED before this
                    lock.wait();                    // WAITING
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }, "Worker");

        System.out.println("After new: " + worker.getState());          // NEW

        worker.start();
        System.out.println("After start: " + worker.getState());        // RUNNABLE

        Thread.sleep(100);
        System.out.println("During sleep: " + worker.getState());       // TIMED_WAITING

        Thread.sleep(600);
        System.out.println("After entering wait: " + worker.getState());// WAITING

        synchronized (lock) {
            lock.notify();
        }

        worker.join();
        System.out.println("After join: " + worker.getState());         // TERMINATED
    }
}
```
### 3.6 Common Methods That Affect Lifecycle
| Method | Effect |
| ----- | ----- |
| `start()`  | NEW → RUNNABLE; creates OS thread, calls `run()`  |
| `run()`  | The code executed by the thread |
| `sleep(ms)`  | Pauses current thread → TIMED_WAITING |
| `yield()`  | Hint to scheduler to give other threads a chance |
| `join()`  | Caller waits for target thread to finish → WAITING/TIMED_WAITING |
| `interrupt()`  | Sets interrupt flag; may wake from WAITING/TIMED_WAITING with `InterruptedException`  |
| `isAlive()`  | True if started and not terminated |
| `setDaemon(true)`  | Marks as daemon; must be done before `start()`  |
| `setPriority(n)`  | Hint (1–10) for scheduler |
### 3.7 Important Subtleties
1. `**sleep**` ** vs **`**wait**` 
    - `sleep`  is a `Thread`  static method; doesn't release lock.
    - `wait`  is on `Object` ; releases the monitor lock and must be called inside `synchronized` .

2. **Once TERMINATED, a thread is dead forever.**t.start();
t.join();
t.start(); // throws IllegalThreadStateException
3. **RUNNABLE doesn't mean "running right now."**
It means the thread is ready to be scheduled. The JVM merges "ready" and "running" into one state.
4. **Spurious wakeups.**
A waiting thread may wake without being notified — always wrap `wait()`  in a `while`  loop:synchronized (lock) {
    while (!condition) lock.wait();
}
5. **Interrupts are cooperative.**
`interrupt()`  doesn't kill a thread — it sets a flag. The thread must check `Thread.interrupted()`  or it'll be thrown via `InterruptedException`  from blocking calls.
6. **A thread can transition between BLOCKED, WAITING, RUNNABLE many times** during its life.
7. **Virtual threads (Java 21+)** use the same `Thread.State`  model but are scheduled by the JVM, not the OS. Their state transitions look the same from the API.
### 3.8 Monitoring Thread States in Production
Tools to inspect live thread states:

- `jstack <pid>`  — text-based thread dump
- `jcmd <pid> Thread.print` 
- **VisualVM**, **JConsole** — GUI
- **Java Flight Recorder (JFR)** — low-overhead profiling
- `ThreadMXBean`  programmatically:
```java
ThreadMXBean bean = ManagementFactory.getThreadMXBean();
for (long id : bean.getAllThreadIds()) {
    ThreadInfo info = bean.getThreadInfo(id);
    System.out.println(info.getThreadName() + " : " + info.getThreadState());
}
```
A **thread dump** is the first thing to look at when diagnosing:

- Deadlocks (`jstack`  automatically reports them)
- Slow responses (threads stuck in BLOCKED/WAITING)
- Thread leaks (too many threads in pool)
### 3.9 Summary Cheat Sheet
| You see thread in… | It likely means… |
| ----- | ----- |
| Many `BLOCKED`  | Lock contention; consider finer-grained locks or lock-free structures |
| Many `WAITING`  | Possibly waiting on a queue or condition; could be fine, or a stuck consumer |
| Many `TIMED_WAITING`  | `sleep`, polling, or pool threads idle on `take()`  |
| Many `RUNNABLE` but low CPU | Threads might be blocked in native I/O (shows as RUNNABLE in Java but actually waiting) |
---

## Summary
- **Concurrency** lets a program make progress on multiple tasks; **parallelism** is doing them literally at the same time.
- **Processes** are isolated, heavyweight OS containers; **threads** are lightweight units sharing memory inside a process.
- Java threads transition through six states: **NEW → RUNNABLE → (BLOCKED / WAITING / TIMED_WAITING) → TERMINATED**.
- Understanding these foundations is essential before tackling synchronization, the Java Memory Model, and higher-level concurrency utilities.




<!--- Eraser file: https://app.eraser.io/workspace/mfQnts0WVgDWoHrtsWoD --->