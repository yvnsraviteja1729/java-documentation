Both **Comparable** and **Comparator** are interfaces used to sort objects in Java, but they serve completely different design purposes.

The easiest way to remember the difference is:

* **Comparable** defines the **default (natural) sorting order** for a class.
* **Comparator** defines **custom (alternate) sorting orders** outside of the class.

---

## High-Level Comparison

| Feature | `Comparable` | `Comparator` |
| --- | --- | --- |
| **Package** | `java.lang` | `java.util` |
| **Method to implement** | `compareTo(Object o)` | `compare(Object o1, Object o2)` |
| **Class Modification** | You **must modify** the actual class to implement it. | You **don't modify** the class; you write it as a separate class, lambda, or anonymous inner class. |
| **Number of sorting strategies** | **Only one** default sorting strategy per class. | **Multiple** sorting strategies (e.g., sort by age, sort by name, sort by salary). |
| **How to trigger sorting** | `Collections.sort(list)` | `Collections.sort(list, new MyComparator())` |

---

## 1. Comparable (Natural Sorting)

Use `Comparable` when a class has a clear, logical default sorting order (like sorting numbers from low to high, or words alphabetically). For example, a `Product` class might naturally sort by its `id`.

### Code Example:

```java
import java.util.*;

// We implement Comparable directly on the class we want to sort
class Product implements Comparable<Product> {
    int id;
    String name;

    public Product(int id, String name) {
        this.id = id;
        this.name = name;
    }

    @Override
    public int compareTo(Product other) {
        // Returns negative if 'this' is smaller, zero if equal, positive if larger
        return Integer.compare(this.id, other.id);
    }

    @Override
    public String toString() { return id + ":" + name; }
}

public class ComparableDemo {
    public static void main(String[] args) {
        List<Product> list = new ArrayList<>(List.of(
            new Product(103, "Laptop"),
            new Product(101, "Phone"),
            new Product(102, "Tablet")
        ));

        // Sorts automatically using the compareTo implementation (by ID)
        Collections.sort(list);
        System.out.println(list); // Output: [101:Phone, 102:Tablet, 103:Laptop]
    }
}

```

---

## 2. Comparator (Custom/Multiple Sorting Strategies)

Use `Comparator` when you cannot modify the source code of the class you are sorting, or when you need **multiple different ways** to sort the same data (e.g., sometimes you want to sort products by price, other times by rating).

### Code Example (Modern Java style using Lambdas):

```java
import java.util.*;

class Employee {
    String name;
    int salary;

    public Employee(String name, int salary) {
        this.name = name;
        this.salary = salary;
    }

    @Override
    public String toString() { return name + "($" + salary + ")"; }
}

public class ComparatorDemo {
    public static void main(String[] args) {
        List<Employee> employees = new ArrayList<>(List.of(
            new Employee("Alice", 90000),
            new Employee("Charlie", 50000),
            new Employee("Bob", 75000)
        ));

        // Strategy 1: Sort by Name using a classic Lambda
        Collections.sort(employees, (e1, e2) -> e1.name.compareTo(e2.name));
        System.out.println("By Name: " + employees); 
        // Output: [Alice($90000), Bob($75000), Charlie($50000)]

        // Strategy 2: Sort by Salary using modern Java Comparator builder methods
        employees.sort(Comparator.comparingInt(e -> e.salary));
        System.out.println("By Salary: " + employees);
        // Output: [Charlie($50000), Bob($75000), Alice($90000)]
    }
}

```

---

## The Contract of `compare` and `compareTo`

Both methods rely on a consistent mathematical return value to determine how elements swap positions during a sorting algorithm (like QuickSort or TimSort):

* **Negative value (`< 0`):** Means the first object is **smaller** than the second object (no swap needed for ascending order).
* **Zero (`0`):** Means both objects are **equal** in terms of sorting priority.
* **Positive value (`> 0`):** Means the first object is **larger** than the second object (the sorting algorithm will swap their order).

> **Pro Tip on Primitive Subtraction:** Avoid doing `this.id - other.id` inside your comparison methods. If one ID is a very large positive number and the other is a very large negative number, the subtraction can overflow the integer boundary (`Integer.MAX_VALUE`), flip signs, and cause silent sorting bugs. Always use built-in helpers like `Integer.compare(a, b)` or `Double.compare(a, b)`.