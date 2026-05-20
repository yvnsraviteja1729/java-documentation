# Relationships in Java (OOP) — Detailed Notes

In Object-Oriented Programming, **relationships** define **how classes/objects connect and interact** with each other. Understanding these is crucial for designing maintainable systems.

---

## 🔹 Types of Relationships in Java

```
Relationships in Java
│
├── 1. IS-A Relationship  →  Inheritance / Implementation
│
└── 2. HAS-A Relationship  →  Association
        │
        ├── Aggregation  (Weak HAS-A)
        └── Composition  (Strong HAS-A)
```

---

# 1️⃣ IS-A Relationship (Inheritance)

Represents **inheritance** — one class is a **type of** another.

Achieved using:
- `extends` keyword (class to class)
- `implements` keyword (class to interface)

### ✅ Example — `extends`
```java
class Animal {
    void eat() {
        System.out.println("Eating...");
    }
}

class Dog extends Animal {       // Dog IS-A Animal
    void bark() {
        System.out.println("Barking...");
    }
}

public class Test {
    public static void main(String[] args) {
        Dog d = new Dog();
        d.eat();    // inherited
        d.bark();   // own method
    }
}
```

➡ **Dog IS-A Animal** — Dog inherits properties & behavior of Animal.

---

### ✅ Example — `implements`
```java
interface Vehicle {
    void start();
}

class Car implements Vehicle {   // Car IS-A Vehicle
    public void start() {
        System.out.println("Car started");
    }
}
```

### ✅ Multi-level Inheritance
```java
class LivingBeing { }
class Animal extends LivingBeing { }
class Dog extends Animal { }     // Dog IS-A Animal IS-A LivingBeing
```

### 🎯 When to Use IS-A
Use when the subclass **truly is a kind of** the superclass.

- ✅ `Car IS-A Vehicle`
- ✅ `Manager IS-A Employee`
- ❌ `Car IS-A Engine` (wrong — Car HAS-A Engine)

---

# 2️⃣ HAS-A Relationship (Association)

When **one class contains a reference** to another class — used to **reuse** functionality without inheritance.

### ✅ Basic Example
```java
class Engine {
    void start() {
        System.out.println("Engine started");
    }
}

class Car {
    Engine engine = new Engine();   // Car HAS-A Engine

    void startCar() {
        engine.start();
        System.out.println("Car is running");
    }
}
```

➡ **Car HAS-A Engine** — Car uses Engine's functionality without inheriting it.

---

## 📌 Association

A **general relationship** between two independent classes. Each has its **own lifecycle**.

### Types based on Multiplicity

| Type | Example |
|------|---------|
| **One-to-One** | Person ↔ Passport |
| **One-to-Many** | Teacher → Students |
| **Many-to-One** | Employees → Department |
| **Many-to-Many** | Students ↔ Courses |

### ✅ Example — One-to-One
```java
class Passport {
    String number;
    Passport(String number) { this.number = number; }
}

class Person {
    String name;
    Passport passport;     // One-to-One association

    Person(String name, Passport passport) {
        this.name = name;
        this.passport = passport;
    }
}
```

### ✅ Example — One-to-Many
```java
class Student {
    String name;
    Student(String name) { this.name = name; }
}

class Teacher {
    String name;
    List<Student> students;   // One Teacher → Many Students
}
```

---

# 🔸 HAS-A: Aggregation vs Composition

Both represent **HAS-A** but differ by **ownership** and **lifecycle dependency**.

| Feature | **Aggregation** | **Composition** |
|---------|----------------|----------------|
| Relationship | Weak HAS-A | Strong HAS-A |
| Lifecycle | Independent | Dependent (child dies with parent) |
| Ownership | Shared | Exclusive |
| UML Symbol | Hollow diamond ◇ | Filled diamond ◆ |
| Example | Department ↔ Professor | Human ↔ Heart |

---

## 🔹 Aggregation (Weak HAS-A)

- Both objects can **exist independently**.
- Child object can be **shared** by multiple parents.

### ✅ Example
```java
class Address {
    String city, state;
    Address(String city, String state) {
        this.city = city;
        this.state = state;
    }
}

class Employee {
    int id;
    String name;
    Address address;          // Aggregation

    Employee(int id, String name, Address address) {
        this.id = id;
        this.name = name;
        this.address = address;
    }

    void display() {
        System.out.println(id + " " + name + " " + address.city + ", " + address.state);
    }
}

public class Test {
    public static void main(String[] args) {
        Address addr = new Address("Pune", "Maharashtra");
        Employee e1 = new Employee(1, "Ankit", addr);
        Employee e2 = new Employee(2, "Ravi", addr);  // shared Address ✅
        e1.display();
        e2.display();
    }
}
```

➡ Even if `Employee` object is destroyed, `Address` can still exist.

---

## 🔹 Composition (Strong HAS-A)

- Child object **cannot exist independently** of the parent.
- If parent is destroyed → child is destroyed too.

### ✅ Example
```java
class Heart {
    void beat() {
        System.out.println("Heart is beating...");
    }
}

class Human {
    private final Heart heart = new Heart();  // Composition

    void live() {
        heart.beat();
        System.out.println("Human is alive");
    }
}

public class Test {
    public static void main(String[] args) {
        Human h = new Human();
        h.live();
        // When Human dies → Heart also dies
    }
}
```

### ✅ Another Example — Car and Engine
```java
class Engine {
    void start() { System.out.println("Engine started"); }
}

class Car {
    private final Engine engine;   // Composition

    Car() {
        engine = new Engine();     // Engine lives only with Car
    }

    void drive() {
        engine.start();
        System.out.println("Car is moving");
    }
}
```

➡ If the `Car` is destroyed, its `Engine` is too — they are tightly bound.

---

# 📊 Complete Comparison Table

| Aspect | **Inheritance (IS-A)** | **Aggregation (HAS-A)** | **Composition (HAS-A)** |
|--------|------------------------|--------------------------|--------------------------|
| Type | IS-A | Weak HAS-A | Strong HAS-A |
| Keyword | `extends` / `implements` | Reference variable | Reference variable |
| Lifecycle | Linked at class level | Independent | Dependent |
| Coupling | Tight | Loose | Tight |
| Reusability | High | High | High |
| Example | Dog → Animal | Employee → Address | Human → Heart |

---

# 🎯 Real-World Examples

| Scenario | Relationship |
|----------|--------------|
| `Manager` extends `Employee` | Inheritance (IS-A) |
| `Library` has multiple `Books` (books exist independently) | Aggregation |
| `House` has `Rooms` (rooms can't exist without house) | Composition |
| `Order` contains many `OrderItems` | Composition |
| `University` has many `Departments` | Composition |
| `Department` has many `Professors` | Aggregation |

---

# 🧠 Choosing the Right Relationship

| Question | Answer → Use |
|----------|-------------|
| Is class B a type of class A? | **Inheritance** |
| Does class A use class B's functionality? | **Association** |
| Can class B exist without class A? | **Aggregation** |
| Does class B's existence depend on class A? | **Composition** |

> 💡 **Rule of Thumb:** *"Favor Composition over Inheritance"* — Composition gives more flexibility and reduces tight coupling.

---

# ✅ Advantages of Using Relationships Properly

- Promotes **code reusability**.
- Makes design more **modular** and **maintainable**.
- Enables **loose coupling** (especially with composition/aggregation).
- Improves **scalability** and **extensibility**.
- Mirrors **real-world models** clearly.

---

# 🚫 Common Mistakes

❌ Using inheritance when composition is more appropriate.
❌ Creating deep inheritance hierarchies (hard to maintain).
❌ Tightly coupling classes when loose coupling suffices.
❌ Confusing aggregation vs composition (always check lifecycle).

---

## 📝 Quick Recap

> - **IS-A** → Inheritance (`extends`, `implements`)
> - **HAS-A** → Association
>   - **Aggregation** = Weak HAS-A (independent lifecycle)
>   - **Composition** = Strong HAS-A (dependent lifecycle)
