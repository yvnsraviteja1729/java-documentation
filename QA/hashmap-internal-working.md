At its core, a **HashMap** in Java operates on the principle of **Hashing**. It maps keys to values using a data structure often referred to as a **bucket array** (an array where each element can hold one or more key-value pairs).

Here is exactly what goes on under the hood when you read, write, or resize a HashMap.

---

## 1. The Core Data Structure

Internally, a HashMap maintains an array of nodes (traditionally called buckets). In the source code, this array is defined as:

```java
transient Node<K,V>[] table;

```

Each `Node<K,V>` is a static inner class that represents a key-value entry and contains four things:

* `int hash`: The calculated hash value of the key.
* `K key`: The actual key object.
* `V value`: The actual value object.
* `Node<K,V> next`: A pointer/reference to the next node (used when a collision occurs).

---

## 2. How `put(K key, V value)` Works Internally

When you call `map.put(key, value)`, Java executes a highly optimized sequence of steps to figure out exactly where that data should live.

### Step A: Calculate the Hash (The Hash Function)

First, Java calls the key's native `hashCode()` method. However, to prevent poor user-defined hash implementations from causing collisions, the HashMap applies a **supplemental hashing function** (called bit-shifting or "shaking"):

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

```

> **What this does:** It takes the 32-bit hash code and XORs (`^`) the higher 16 bits with the lower 16 bits (`>>> 16`). This spreads the higher bits downwards, ensuring that even if hash codes differ only in their upper bits, they will still distribute evenly across the array.
> *Note: A `null` key always maps to hash `0`, meaning it always lands in bucket index `0`.*

### Step B: Compute the Bucket Index

The JVM needs to convert that huge 32-bit hash value into a valid index within the current array size (default initial size is 16). Instead of using the slower modulo (`%`) operator, Java uses a fast bitwise **AND** operation:

$$\text{index} = \text{hash} \ \& \ (n - 1)$$

*(Where $n$ is the current length of the array).*
Because HashMap array capacities are **always a power of 2**, $(n - 1)$ acts as a perfect bitmask (e.g., if $n = 16$, $n-1 = 15$, which is `00001111` in binary). This isolates the lowest bits and instantly yields an index within bounds.

### Step C: Handle Collisions

Once the index is calculated, the HashMap checks that bucket:

1. **Bucket is Empty:** A new `Node` is created and placed directly into that array slot.
2. **Collision Occurs (Bucket is occupied):** If a node already exists at that index, Java checks if the key matches. If the keys are identical (using `.equals()`), the old value is overwritten. If the keys are different, it walks down the chain:
* **Linked List:** By default, it appends the new node to the end of a Singly Linked List (this approach is called *Separate Chaining*).
* **Balanced Tree (Treeification):** If the linked list grows too long (**8 or more nodes** in a single bucket) **AND** the total map capacity is at least **64**, the HashMap converts that specific linked list into a **Red-Black Tree** (changing the nodes from `Node` to `TreeNode`). This prevents a worst-case performance degradation from $O(1)$ down to $O(n)$, capping it at $O(\log n)$.



---

## Visualizing the Internal Structure

This architectural layout shows how a single bucket array can branch out into either linear linked lists or balanced trees depending on collision depth:

```
Bucket Array (table)
+---+
| 0 | --> [ Hash | Null Key | Value ]
+---+
| 1 | --> Empty
+---+
| 2 | --> [ Hash | KeyA | Value ] ----> [ Hash | KeyB | Value ]  (Linked List)
+---+
| 3 | --> Empty
+---+
| 4 | --> [ TreeNode (Root) ]
|   |        /          \
|   |   [TreeNode]   [TreeNode]                                 (Red-Black Tree)
+---+

```

---

## 3. How `get(Object key)` Works Internally

Retrieving a value is a straightforward mirror image of the insertion process:

1. Calculate the `hash(key)`.
2. Compute the target bucket index using `hash & (n - 1)`.
3. Go to that index in the array.
4. **Compare the keys:** First, it checks the root node of that bucket. If `(node.hash == hash && (node.key == key || node.key.equals(key)))`, it returns that node's value.
5. If it doesn't match the first node, it checks if the bucket has been converted to a tree. If yes, it searches the Red-Black Tree in $O(\log n)$ time. If no, it traverses the linked list sequentially using `.next` in $O(n)$ time until it finds a match or hits `null`.

---

## 4. Resizing and the Load Factor

To maintain constant time performance ($O(1)$) for reads and writes, a HashMap cannot let its buckets get too crowded. It uses two key parameters to determine when to expand:

* **Initial Capacity (Default: 16):** The starting number of buckets in the array.
* **Load Factor (Default: 0.75):** The threshold percentage of fullness.

### The Threshold Calculation

$$\text{Threshold} = \text{Capacity} \times \text{Load Factor}$$

With defaults, $\text{Threshold} = 16 \times 0.75 = 12$. As soon as you add the **13th** item to the HashMap, a process called **rehash / resize** triggers.

### What happens during a Resize?

1. The capacity of the array **doubles** (e.g., from 16 to 32).
2. A brand-new array is allocated.
3. **The Rehashing Phase:** Because the array length ($n$) has changed, the bitmask $(n-1)$ has also changed. Every single existing element in the map must have its bucket index recalculated.
4. The elements are distributed into the new array blocks. In Java 8, this is highly optimized: because the size doubles, an element either stays at its **exact same index** or moves ahead by exactly the **old capacity amount** (index + oldCap).

---

## Key Takeaway: Why Immutable Keys Matter

This entire mechanism depends heavily on the assumption that a key's `hashCode()` will **never change** while it is inside the map.

If you use a mutable object as a key (like a custom object or an `ArrayList`) and modify its internal properties after putting it in the map, its `hashCode()` changes. If you try to call `get()` later, the HashMap will compute the *new* hash, point to the wrong bucket index, and fail to find your object—creating a silent, permanent memory leak. This is why classes like `String` and `Integer` make perfect HashMap keys; they are completely immutable.</K,V></K,V>