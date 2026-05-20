In Java, memory management is divided into two primary zones: the **Stack** and the **Heap**. They serve entirely different purposes, operate on different data structures, and have unique lifecycles.

Here is a breakdown of how they work, what they store, and a concrete code example to visualize it.

---

## The High-Level Difference

| Feature | Stack Memory | Heap Memory |
| --- | --- | --- |
| **What it stores** | Local variables, primitive data types, and references (pointers) to objects. | Actual objects and instance variables. |
| **Access Speed** | **Very Fast** (LIFO - Last In, First Out structure). | **Slower** (Requires pointer lookups and dynamic allocation). |
| **Lifecycle** | Tied to the execution of a method. Memory is cleared as soon as the method finishes. | Tied to the application. Objects stay until they are cleaned up by the **Garbage Collector**. |
| **Scope** | Private to the specific thread executing the code. | Shared across all threads in the application. |
| **Size Limit** | Small and fixed (Can throw `StackOverflowError`). | Large and dynamic (Can throw `OutOfMemoryError`). |

---

## What Information is Stored Where?

### 1. Stack Memory

Think of the stack as the "temporary workspace" for a running thread. Every time a method is called, a new **stack frame** is created. It stores:

* **Primitive Local Variables:** Values of types like `int`, `double`, `boolean`, `char`, etc., defined inside a method.
* **Object References:** The "address" or "pointer" pointing to where an actual object lives on the heap.
* **Method Execution Flow:** Information about which method called which, tracking where to return when the current method completes.

### 2. Heap Memory

Think of the heap as the "main warehouse." It is a massive pool of memory where all dynamic data lives. It stores:

* **All Objects:** Any entity created using the `new` keyword (e.g., `new String()`, `new ArrayList()`, or custom objects).
* **Instance Variables:** Fields belonging to an object (even if they are primitives like `int id`), because they must live inside the object itself.

---

## Conceptual Architecture

As the diagram implies, the stack holds the immediate, fast-access instructions and pointers, while the heap holds the heavy, structured data blocks.

---

## Detailed Example

Let’s trace a simple Java program to see exactly how memory is allocated between the Stack and the Heap.

```java
public class MemoryDemo {
    public static void main(String[] args) {
        int age = 25;                       // Line 1
        Order currentOrder = new Order(101);// Line 2
        process(currentOrder);              // Line 3
    }

    public static void process(Order order) {
        int bonus = 5;                      // Line 4
        // Do something...
    }
}

class Order {
    int id;
    public Order(int id) { this.id = id; }
}

```

### Memory Allocation Step-by-Step:

1. **Line 1 (`int age = 25;`):**
A stack frame for `main()` is created. The primitive variable `age` and its literal value `25` are stored directly inside this stack frame.
2. **Line 2 (`Order currentOrder = new Order(101);`):**
* The `new Order(101)` part allocates space on the **Heap**. Inside this heap space, the object's instance variable `id = 101` is stored.
* The variable `currentOrder` is created in the `main()` **Stack frame**. It doesn't hold the order data; it holds the *memory address* (e.g., `0x7a4b`) pointing to that object on the heap.


3. **Line 3 & 4 (`process(currentOrder);`):**
When `process()` is called, a *new* stack frame for `process()` is pushed on top of the stack.
* A copy of the reference address (`0x7a4b`) is passed into the `order` parameter inside the new stack frame. Both `currentOrder` (in main) and `order` (in process) now point to the exact same object on the heap.
* The local primitive `bonus = 5` is added to the `process()` stack frame.



### Memory Cleanup:

* As soon as the `process()` method finishes, its entire stack frame (including `bonus` and the local `order` reference) is **instantly popped and destroyed**.
* When `main()` ends, its stack frame is cleared.
* The `Order` object on the heap is now left with zero references pointing to it. It becomes "eligible for Garbage Collection" and will eventually be wiped from the heap by the JVM later.

---

## Why this Distinction Matters

Understanding this prevents bugs like the `NullPointerException` (having a reference variable on the stack that points to `null` instead of a valid heap address) and helps you design memory-efficient applications by knowing when objects are being duplicated versus when they are simply being pointed to by multiple references.