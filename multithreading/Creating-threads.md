<p><a target="_blank" href="https://app.eraser.io/workspace/cSEI99AeSpK1R076tKJQ" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>

This document covers **every standard way** to create and run threads in Java — from the original `Thread` class to modern **virtual threads** (Java 21+) and **structured concurrency**. Each approach includes runnable examples, pros/cons, and when to use them.

---

## Overview: All the Ways to Create Threads in Java
| # | Approach | Java Version | Typical Use Case |
| ----- | ----- | ----- | ----- |
| 1 | Extending `Thread` class | 1.0 | Learning / rare cases |
| 2 | Implementing `Runnable`  | 1.0 | Simple background tasks |
| 3 | Implementing `Callable` + `FutureTask`  | 1.5 | Tasks that return a value or throw |
| 4 | `ExecutorService` + `Runnable`/`Callable`  | 1.5 | Production: thread reuse via pools |
| 5 | `ScheduledExecutorService`  | 1.5 | Periodic / delayed tasks |
| 6 | `ForkJoinPool` + `RecursiveTask`/`RecursiveAction`  | 1.7 | Divide-and-conquer parallelism |
| 7 | `CompletableFuture.supplyAsync` / `runAsync`  | 1.8 | Async pipelines, composition |
| 8 | `Thread.Builder` (modern factory) | 19+ | Cleaner thread creation |
| 9 | **Virtual Threads** | 21+ | Massive I/O concurrency |
| 10 | **Structured Concurrency** (`StructuredTaskScope`) | 21 (preview) / 25 | Grouped, cancellable subtasks |
| 11 | `TimerTask` / `Timer`  | 1.3 | Legacy scheduling (avoid) |
| 12 | Custom `ThreadFactory`  | 1.5 | Naming/configuring pool threads |
---

## 1. Extending the `Thread` Class
The most direct (and oldest) way: subclass `Thread` and override `run()`.

```java
public class MyThread extends Thread {

    public MyThread(String name) {
        super(name);                          // give the thread a name
    }

    @Override
    public void run() {
        for (int i = 1; i <= 3; i++) {
            System.out.println(getName() + " - iteration " + i);
            try { Thread.sleep(200); } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }

    public static void main(String[] args) {
        MyThread t1 = new MyThread("Worker-1");
        MyThread t2 = new MyThread("Worker-2");

        t1.start();   // schedules run() on a NEW thread
        t2.start();
        // NEVER call t1.run() directly — it would execute on the main thread
    }
}
```
### Key Points
- `start()`  → creates a new OS thread that calls `run()` .
- `run()`  → just an ordinary method if called directly (no new thread).
- A thread can be started only once (`IllegalThreadStateException`  otherwise).
### Pros / Cons
✅ Simple, no extra interfaces.
❌ Couples task logic with thread management.
❌ Java only supports single inheritance — extending `Thread` blocks extending any other class.
❌ Hard to reuse the same task on multiple threads.

>  **Generally discouraged.** Prefer `Runnable` or higher-level abstractions. 

---

## 2. Implementing the `Runnable` Interface
`Runnable` separates the **task** from the **thread that runs it**.

```java
public class RunnableExample implements Runnable {

    @Override
    public void run() {
        System.out.println("Running in: " + Thread.currentThread().getName());
    }

    public static void main(String[] args) {
        Runnable task = new RunnableExample();
        Thread t = new Thread(task, "Worker-1");
        t.start();
    }
}
```
### Same Thing with a Lambda (Java 8+)
`Runnable` is a functional interface, so:

```java
public class RunnableLambda {
    public static void main(String[] args) {
        Runnable task = () -> {
            for (int i = 1; i <= 3; i++) {
                System.out.println(Thread.currentThread().getName() + " - " + i);
            }
        };

        new Thread(task, "Worker-A").start();
        new Thread(task, "Worker-B").start();   // same task, two threads
    }
}
```
### Pros / Cons
✅ Decouples task from thread.
✅ Same `Runnable` can be submitted to many threads/executors.
✅ Class can still extend another class.
❌ Cannot return a result or throw a checked exception.

>  **Use **`**Runnable**`** for fire-and-forget tasks.** 

---

## 3. Implementing `Callable<V>` with `FutureTask` 
`Callable<V>` is like `Runnable`, but it:

- **Returns** a value of type `V` .
- Can **throw a checked exception**.
```java
import java.util.concurrent.*;

public class CallableExample {
    public static void main(String[] args) throws Exception {

        Callable<Integer> task = () -> {
            Thread.sleep(500);
            return 42;
        };

        FutureTask<Integer> futureTask = new FutureTask<>(task);
        Thread t = new Thread(futureTask, "Computer");
        t.start();

        System.out.println("Result: " + futureTask.get());  // blocks until done
    }
}
```
### Key Points
- `FutureTask`  implements **both** `Runnable`  and `Future<V>` .
- `get()`  **blocks** until the result is ready.
- `get(timeout, unit)`  provides a timed wait.
- `cancel(true)`  attempts to interrupt the running task.
### Pros / Cons
✅ Tasks can return values and throw checked exceptions.
✅ Supports cancellation and timed waits via `Future`.
❌ Still manual thread management (in most real code you'd use an executor).

>  **Use when you need a result back from the thread.** 

---

## 4. Executor Framework: `ExecutorService` 
The recommended way to run threads in production. **Decouples task submission from thread management** — the executor maintains and reuses threads.

```java
import java.util.concurrent.*;

public class ExecutorBasic {
    public static void main(String[] args) throws Exception {

        ExecutorService pool = Executors.newFixedThreadPool(3);

        // 1. Submit Runnable (no return value)
        pool.submit(() -> System.out.println("Runnable on " + Thread.currentThread().getName()));

        // 2. Submit Callable (returns a Future)
        Future<Integer> future = pool.submit(() -> {
            Thread.sleep(300);
            return 7 * 6;
        });
        System.out.println("Callable result: " + future.get());

        // 3. invokeAll — run many callables, wait for all
        var tasks = java.util.List.of(
                (Callable<String>) () -> "A",
                (Callable<String>) () -> "B",
                (Callable<String>) () -> "C");
        for (Future<String> f : pool.invokeAll(tasks)) {
            System.out.println(f.get());
        }

        // 4. invokeAny — run many, return the first successful result
        String first = pool.invokeAny(tasks);
        System.out.println("First: " + first);

        pool.shutdown();   // disallow new tasks; existing tasks finish
        pool.awaitTermination(2, TimeUnit.SECONDS);
    }
}
```
### Factory Methods on `Executors` 
| Method | Description |
| ----- | ----- |
| `newFixedThreadPool(n)`  | Fixed n threads (unbounded queue ⚠️) |
| `newCachedThreadPool()`  | Grows as needed (unbounded thread count ⚠️) |
| `newSingleThreadExecutor()`  | One worker, sequential execution |
| `newScheduledThreadPool(n)`  | Delayed / periodic tasks |
| `newWorkStealingPool()`  | Parallelism based on CPU cores; uses ForkJoinPool |
| `newVirtualThreadPerTaskExecutor()`  | Java 21+: one virtual thread per task |
For production, prefer constructing `**ThreadPoolExecutor**`** directly** with a **bounded queue**:

```java
ThreadPoolExecutor pool = new ThreadPoolExecutor(
    4,                                  // corePoolSize
    16,                                 // maximumPoolSize
    60, TimeUnit.SECONDS,               // idle keep-alive
    new ArrayBlockingQueue<>(500),      // bounded queue
    Executors.defaultThreadFactory(),
    new ThreadPoolExecutor.CallerRunsPolicy()  // backpressure
);
```
### Shutdown Pattern
```java
pool.shutdown();                          // no new tasks
try {
    if (!pool.awaitTermination(30, TimeUnit.SECONDS)) {
        pool.shutdownNow();               // force cancel
    }
} catch (InterruptedException e) {
    pool.shutdownNow();
    Thread.currentThread().interrupt();
}
```
Or with try-with-resources (Java 19+, since `ExecutorService` is now `AutoCloseable`):

```java
try (ExecutorService pool = Executors.newFixedThreadPool(4)) {
    pool.submit(() -> doWork());
}   // auto-shutdown + awaitTermination
```
### Pros / Cons
✅ Reuses threads → avoids creation overhead.
✅ Built-in queueing, lifecycle, rejection policies.
✅ Works with both `Runnable` and `Callable`.
❌ Misconfiguration (unbounded queue/pool) can cause OOM.

>  **The default choice for production code.** 

---

## 5. `ScheduledExecutorService` — Delayed and Periodic Tasks
For tasks that run after a delay or repeat at a fixed rate.

```java
import java.util.concurrent.*;

public class ScheduledExample {
    public static void main(String[] args) throws Exception {

        ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(2);

        // One-shot delayed task
        scheduler.schedule(() -> System.out.println("After 1 second"),
                           1, TimeUnit.SECONDS);

        // Run repeatedly, regardless of how long each run takes
        scheduler.scheduleAtFixedRate(
            () -> System.out.println("Tick at " + System.currentTimeMillis()),
            0, 500, TimeUnit.MILLISECONDS);

        // Run repeatedly, with fixed gap AFTER each task ends
        scheduler.scheduleWithFixedDelay(
            () -> System.out.println("After-delay tick"),
            0, 500, TimeUnit.MILLISECONDS);

        Thread.sleep(3000);
        scheduler.shutdown();
    }
}
```
### `scheduleAtFixedRate` vs `scheduleWithFixedDelay` 
| Method | Behavior |
| ----- | ----- |
| `scheduleAtFixedRate`  | Next run starts at `start + period`, regardless of previous run's duration. If a run is slow, runs can pile up. |
| `scheduleWithFixedDelay`  | Next run starts `delay` after the **previous run ended**. Safer when task time varies. |
>  Prefer `scheduleWithFixedDelay` unless you specifically need a fixed cadence. 

---

## 6. `ForkJoinPool` + `RecursiveTask` / `RecursiveAction` 
For **divide-and-conquer** parallelism. Each worker has its own deque; idle workers **steal** tasks from others.

```java
import java.util.concurrent.RecursiveTask;
import java.util.concurrent.ForkJoinPool;

public class ForkJoinSum extends RecursiveTask<Long> {

    private static final int THRESHOLD = 10_000;
    private final long[] arr;
    private final int lo, hi;

    public ForkJoinSum(long[] arr, int lo, int hi) {
        this.arr = arr; this.lo = lo; this.hi = hi;
    }

    @Override
    protected Long compute() {
        if (hi - lo <= THRESHOLD) {
            long sum = 0;
            for (int i = lo; i < hi; i++) sum += arr[i];
            return sum;
        }
        int mid = (lo + hi) >>> 1;
        ForkJoinSum left  = new ForkJoinSum(arr, lo, mid);
        ForkJoinSum right = new ForkJoinSum(arr, mid, hi);
        left.fork();                  // run async
        long r = right.compute();     // run inline
        long l = left.join();         // wait for the forked half
        return l + r;
    }

    public static void main(String[] args) {
        long[] data = new long[1_000_000];
        for (int i = 0; i < data.length; i++) data[i] = i;

        long sum = ForkJoinPool.commonPool().invoke(new ForkJoinSum(data, 0, data.length));
        System.out.println("Sum = " + sum);
    }
}
```
- Use `RecursiveAction`  if there's no return value.
- Used internally by `parallelStream()`  and `CompletableFuture`  (common pool).
>  **Use for CPU-bound divide-and-conquer.** Avoid blocking inside tasks. 

---

## 7. `CompletableFuture` — Async Tasks Without Manual Threads
Modern approach for asynchronous, composable computations. Uses `ForkJoinPool.commonPool()` by default, or your own executor.

```java
import java.util.concurrent.*;

public class CompletableFutureExample {
    public static void main(String[] args) throws Exception {

        // Runs Runnable asynchronously (no result)
        CompletableFuture<Void> r = CompletableFuture.runAsync(
            () -> System.out.println("Running on " + Thread.currentThread().getName()));

        // Runs Supplier asynchronously (returns a result)
        CompletableFuture<Integer> s = CompletableFuture.supplyAsync(() -> {
            try { Thread.sleep(200); } catch (InterruptedException e) {}
            return 21;
        });

        // Chain transformations
        CompletableFuture<Integer> doubled = s.thenApply(x -> x * 2);

        System.out.println("Result: " + doubled.get());   // 42
        r.get();   // wait for fire-and-forget too
    }
}
```
### With a Custom Executor
```java
ExecutorService executor = Executors.newFixedThreadPool(4);
CompletableFuture.supplyAsync(() -> fetch(), executor)
                 .thenApplyAsync(this::transform, executor)
                 .thenAccept(System.out::println);
```
>  Provide your own executor for **I/O-bound** work; don't pollute the common pool. 

---

## 8. `Thread.Builder` (Java 19+)
A cleaner, fluent API to configure threads (and in particular, **virtual threads**).

```java
public class ThreadBuilderExample {
    public static void main(String[] args) throws Exception {

        // Platform (OS) thread builder
        Thread platform = Thread.ofPlatform()
                                .name("worker-", 0)         // worker-0, worker-1, …
                                .daemon(true)
                                .priority(Thread.NORM_PRIORITY)
                                .start(() -> System.out.println("Hello from platform thread"));

        // Virtual thread builder (Java 21+)
        Thread virtual = Thread.ofVirtual()
                               .name("vt-worker")
                               .start(() -> System.out.println("Hello from virtual thread"));

        platform.join();
        virtual.join();
    }
}
```
You can also create a `**ThreadFactory**` from a builder:

```java
ThreadFactory factory = Thread.ofVirtual().name("api-", 0).factory();
ExecutorService es = Executors.newThreadPerTaskExecutor(factory);
```
---

## 9. Virtual Threads (Java 21+)
Lightweight threads managed by the JVM. Cheap enough to create **millions** of them. Best for **I/O-bound** workloads (HTTP, DB, RPC).

### a) Direct creation
```java
public class VirtualThreadDirect {
    public static void main(String[] args) throws Exception {
        Thread vt = Thread.startVirtualThread(() ->
            System.out.println("Hello from " + Thread.currentThread()));
        vt.join();
    }
}
```
### b) Using `Thread.ofVirtual()` 
```java
Thread vt = Thread.ofVirtual()
                  .name("user-handler-1")
                  .unstarted(() -> handle());
vt.start();
```
### c) Per-task virtual thread executor (recommended)
```java
import java.util.concurrent.Executors;
import java.util.stream.IntStream;

public class VirtualThreadExecutor {
    public static void main(String[] args) {
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            IntStream.range(0, 10_000).forEach(i ->
                executor.submit(() -> {
                    Thread.sleep(1000);          // would never scale on OS threads
                    System.out.println("done " + i);
                    return null;
                }));
        }   // try-with-resources auto-closes
    }
}
```
### Caveats
- Don't **pool** virtual threads — they're already cheap; create per task.
- Use `ReentrantLock`  instead of `synchronized`  for long blocking operations to avoid **pinning** the carrier thread.
- Don't use them for CPU-bound work (use platform threads / ForkJoinPool).
---

## 10. Structured Concurrency (Java 21 preview, stabilizing in 25)
Treat a group of related subtasks as **one unit of work** — with bounded lifetime and clean cancellation.

```java
import java.util.concurrent.StructuredTaskScope;

public class StructuredConcurrencyExample {

    record User(String name) {}
    record Order(int id) {}

    static User  fetchUser()  throws InterruptedException { Thread.sleep(200); return new User("Ada"); }
    static Order fetchOrder() throws InterruptedException { Thread.sleep(300); return new Order(42); }

    public static void main(String[] args) throws Exception {

        try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {

            var userTask  = scope.fork(() -> fetchUser());
            var orderTask = scope.fork(() -> fetchOrder());

            scope.join();             // wait for both
            scope.throwIfFailed();    // propagate any error

            System.out.println(userTask.get() + " / " + orderTask.get());
        }
    }
}
```
If one subtask fails, the scope automatically **cancels** the others. Perfect for fan-out/aggregation.

>  **The future of writing concurrent Java.** Use with virtual threads for clean, scalable code. 

---

## 11. Legacy: `Timer` + `TimerTask` 
Old scheduling API from Java 1.3. **Avoid in new code** — single-threaded, unhandled exceptions kill the timer.

```java
import java.util.Timer;
import java.util.TimerTask;

public class TimerExample {
    public static void main(String[] args) {
        Timer timer = new Timer("scheduler", true);
        timer.scheduleAtFixedRate(new TimerTask() {
            public void run() { System.out.println("Tick"); }
        }, 0, 1000);
    }
}
```
>  Use `ScheduledExecutorService` instead. 

---

## 12. Custom `ThreadFactory` 
When you submit tasks to an executor, the executor creates threads through a `ThreadFactory`. A custom one lets you set names, priorities, daemon flag, and uncaught exception handlers.

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class NamedThreadFactory implements ThreadFactory {

    private final String prefix;
    private final boolean daemon;
    private final AtomicInteger idx = new AtomicInteger(1);

    public NamedThreadFactory(String prefix, boolean daemon) {
        this.prefix = prefix; this.daemon = daemon;
    }

    @Override
    public Thread newThread(Runnable r) {
        Thread t = new Thread(r, prefix + "-" + idx.getAndIncrement());
        t.setDaemon(daemon);
        t.setUncaughtExceptionHandler((th, ex) ->
            System.err.println("Uncaught in " + th.getName() + ": " + ex));
        return t;
    }

    public static void main(String[] args) {
        ExecutorService pool = Executors.newFixedThreadPool(
            4, new NamedThreadFactory("api-worker", false));
        pool.submit(() -> System.out.println(Thread.currentThread().getName()));
        pool.shutdown();
    }
}
```
>  **Always name your threads in production** — it makes thread dumps and logs vastly easier to read. 

---

## Bonus: Important `Thread` Class Methods (Cheat Sheet)
| Method | Purpose |
| ----- | ----- |
| `start()`  | Starts a new thread that runs `run()`  |
| `run()`  | The code the thread executes |
| `sleep(ms)`  | <p>Static — pauses the current thread (does </p><p>**not**</p><p> release locks)</p> |
| `yield()`  | Static — hint to scheduler to give others a chance |
| `join()`  | Wait for this thread to finish |
| `join(ms)`  | Wait with timeout |
| `interrupt()`  | Sets the interrupt flag; wakes from blocking calls with `InterruptedException`  |
| `isInterrupted()`  | Returns whether the flag is set |
| `Thread.interrupted()`  | <p>Static — returns the flag </p><p>**and clears it**</p> |
| `setName(s)`  | Set thread name |
| `setDaemon(true)`  | Mark daemon (must be before `start`) |
| `setPriority(n)`  | Hint priority (1–10) |
| `getState()`  | Current `Thread.State`  |
| `currentThread()`  | Static — current executing thread |
| `setUncaughtExceptionHandler(h)`  | Handle uncaught exceptions |
---

## Choosing the Right Approach — Decision Guide
| If you need to… | Use… |
| ----- | ----- |
| Fire a quick background task | `new Thread(Runnable).start()` (only in toy code) |
| Run a task that returns a value | `Callable` + `ExecutorService.submit`  |
| Run many tasks efficiently | `ExecutorService` (`newFixedThreadPool` / custom `ThreadPoolExecutor`) |
| Run delayed or periodic tasks | `ScheduledExecutorService`  |
| Parallelize CPU-bound divide-and-conquer | `ForkJoinPool` + `RecursiveTask`  |
| Compose async pipelines | `CompletableFuture`  |
| Handle massive concurrent I/O | **Virtual threads** (`newVirtualThreadPerTaskExecutor`) |
| Group related concurrent subtasks | **Structured concurrency** (`StructuredTaskScope`) |
| Configure thread names/daemon | Custom `ThreadFactory` or `Thread.Builder`  |
---

## Best Practices
✅ **Do**

- Prefer **executors** over raw `new Thread()` .
- Always **name** your threads (custom `ThreadFactory`  or `Thread.Builder` ).
- Use **bounded** queues in `ThreadPoolExecutor` ; set an appropriate rejection policy.
- Always **shut down** executors (`shutdown()`  + `awaitTermination()` ), or use try-with-resources.
- Restore the interrupt flag when catching `InterruptedException` :catch (InterruptedException e) { Thread.currentThread().interrupt(); }
- Use **virtual threads** for blocking I/O at scale.
- Use **structured concurrency** for related subtasks.
❌ **Don't**

- Don't call `run()`  directly — it doesn't create a new thread.
- Don't use `Thread.stop()` , `suspend()` , `resume()`  — deprecated and unsafe.
- Don't use unbounded `Executors.newCachedThreadPool()`  in production.
- Don't ignore `InterruptedException` .
- Don't pool virtual threads.
- Don't pin virtual threads with `synchronized`  around blocking I/O — use `ReentrantLock` .
- Don't rely on thread priorities for correctness.
---

## Summary
Java offers a **spectrum** of thread creation mechanisms from low-level (`new Thread(...)`) to high-level (`CompletableFuture`, virtual threads, structured concurrency). The right choice depends on:

- **Workload type** — CPU-bound vs I/O-bound.
- **Scale** — a few threads vs millions.
- **Composition** — single task vs pipeline vs grouped subtasks.
- **Result handling** — fire-and-forget vs value-returning vs async.
>  **Modern rule of thumb:** Use `ExecutorService` + `CompletableFuture` for most apps; switch to **virtual threads + structured concurrency** when targeting Java 21+ and dealing with I/O-heavy workloads. 







<!--- Eraser file: https://app.eraser.io/workspace/cSEI99AeSpK1R076tKJQ --->