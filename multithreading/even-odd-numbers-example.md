<p><a target="_blank" href="https://app.eraser.io/workspace/IWCBQ1jzII1Nrf57vON6" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>

To coordinate two threads so they print in the strict order of $1, 2, 3, \dots, 100$, we use a shared **lock object** and the `wait()` / `notify()` mechanism. This ensures the threads "hand off" control to each other rather than racing to the console.

### The Java Solution
Java

```
public class NumberPrinter {
    private final int limit = 100;
    private int counter = 1;
    private final Object lock = new Object();

    public void printOdd() {
        synchronized (lock) {
            while (counter <= limit) {
                if (counter % 2 == 0) {
                    try {
                        lock.wait();
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    }
                } else {
                    System.out.println(Thread.currentThread().getName() + ": " + counter);
                    counter++;
                    lock.notify();
                }
            }
        }
    }

    public void printEven() {
        synchronized (lock) {
            while (counter <= limit) {
                if (counter % 2 != 0) {
                    try {
                        lock.wait();
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    }
                } else {
                    System.out.println(Thread.currentThread().getName() + ": " + counter);
                    counter++;
                    lock.notify();
                }
            }
        }
    }

    public static void main(String[] args) {
        NumberPrinter printer = new NumberPrinter();

        Thread t1 = new Thread(printer::printOdd, "Odd-Thread");
        Thread t2 = new Thread(printer::printEven, "Even-Thread");

        t1.start();
        t2.start();
    }
}
```
---

### How it Works
- **The Lock Object:** Both threads synchronize on the same `lock`  instance. Only one thread can enter its block at a time.
- **The Wait/Notify Loop:**
    - If the **Odd-Thread** sees the counter is even, it calls `wait()` , releasing the lock and "sleeping."
    - Once the **Even-Thread** prints a number, it calls `notify()` , which wakes up the Odd-Thread to check the condition again.

- **Thread Safety:** The `counter`  variable is shared, but because of the `synchronized`  block, we avoid "race conditions" where both threads might try to increment it simultaneously.
### Pro-Tip
While `notify()` works here because we only have two threads, in more complex multi-threaded environments, it is generally safer to use `notifyAll()` to ensure the correct waiting thread is signaled.



<!--- Eraser file: https://app.eraser.io/workspace/IWCBQ1jzII1Nrf57vON6 --->