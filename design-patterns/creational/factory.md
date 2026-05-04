<p><a target="_blank" href="https://app.eraser.io/workspace/VzcooOTNTkbg2UBy9oOV" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>

The **Factory Method Pattern** is a creational design pattern that provides an interface for creating objects in a superclass but allows subclasses to alter the type of objects that will be created.

Instead of calling `new` directly to instantiate an object (which couples your code to a specific class), you call a "factory method" to do it for you.

---

### The Core Structure
To understand this pattern, think of it in four parts:

1. **Product Interface**: The common interface for all objects the factory can create.
2. **Concrete Products**: The actual implementations of that interface.
3. **Creator (Base Class)**: Declares the factory method. It often contains business logic that relies on the Product.
4. **Concrete Creators**: Subclasses that override the factory method to return a specific Concrete Product.
---

### Real-World Example: A Logistics System
Imagine you are building a logistics management application. Initially, it only handles **Truck** transport. However, you later want to add **Ships**.

If you hardcoded `new Truck()` everywhere, adding ships would require changing the code in dozens of places. The Factory Method solves this.

#### 1. The Product Interface
Java

```
public interface Transport {
    void deliver();
}
```
#### 2. Concrete Products
Java

```
public class Truck implements Transport {
    @Override
    public void deliver() {
        System.out.println("Delivering by land in a box truck.");
    }
}

public class Ship implements Transport {
    @Override
    public void deliver() {
        System.out.println("Delivering by sea in a container ship.");
    }
}
```
#### 3. The Creator (The "Factory" Holder)
Java

```
public abstract class Logistics {
    // This is the core "Factory Method"
    public abstract Transport createTransport();

    // The Creator also contains "Business Logic"
    public void planDelivery() {
        Transport t = createTransport();
        t.deliver();
    }
}
```
#### 4. Concrete Creators
Java

```
public class RoadLogistics extends Logistics {
    @Override
    public Transport createTransport() {
        return new Truck();
    }
}

public class SeaLogistics extends Logistics {
    @Override
    public Transport createTransport() {
        return new Ship();
    }
}
```
---

### When to Use This?
- **Decoupling**: When you don't know the exact types and dependencies of the objects your code should work with.
- **Extensibility**: When you want to provide users of your library or framework with a way to extend its internal components.
- **Resource Management**: When you want to save system resources by reusing existing objects instead of rebuilding them each time (though this is often combined with a pool).
### Why use it over a Simple Factory?
A "Simple Factory" is usually just one class with a big `switch` statement. While easy, it violates the **Open/Closed Principle**. If you add a new transport type, you have to modify the factory class.

With the **Factory Method Pattern**, you simply create a new subclass (e.g., `AirLogistics`) without touching any of the existing code. It’s cleaner, safer, and much more scalable.



<!--- Eraser file: https://app.eraser.io/workspace/VzcooOTNTkbg2UBy9oOV --->