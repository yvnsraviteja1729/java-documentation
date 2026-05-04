<p><a target="_blank" href="https://app.eraser.io/workspace/9DAaxO05AuRKxTiofNbK" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>

The **Prototype Pattern** is a creational design pattern that allows you to create new objects by **copying an existing object** (the prototype) rather than creating them from scratch using a constructor.

Think of it as the "Copy-Paste" of software development. Instead of calling `new MyObject()` and manually setting 50 different fields, you take a "Master Copy" that is already configured and simply clone it.

---

### Why use the Prototype Pattern?
1. **Expensive Creation:** Sometimes creating an object is "heavy." It might involve complex database queries, network calls, or intensive calculations. Cloning an existing object in memory is significantly faster.
2. **State Complexity:** If you have an object that is already in a specific, complex state (e.g., a game character with specific armor, health, and inventory), it's easier to clone that character than to rebuild that state from zero.
3. **Avoiding Subclassing:** It allows you to produce new instances without being coupled to the concrete classes of the objects you are duplicating.
---

### Real-World Example: Game NPCs
Imagine a game like _Skyrim_. You need to spawn 100 "Forest Bandits." Every bandit has the same basic stats, a standard leather armor set, and an iron sword.

Instead of running a script for each bandit to:

1. `new Bandit()` 
2. `loadArmorTextures()` 
3. `calculateInitialStats()` 
4. `attachWeapon()` 
You create one **Prototype Bandit** during the game's loading screen. When a player enters a forest, you simply **clone** that prototype 100 times. Each clone can then be slightly modified (e.g., changing their hair color) without re-running the expensive initialization logic.

---

### Implementation in Java
In Java, this is typically handled by implementing the `Cloneable` interface and overriding the `clone()` method.

Java

```
import java.util.ArrayList;
import java.util.List;

// The Prototype
public class Document implements Cloneable {
    private String header;
    private List<String> content = new ArrayList<>();

    public Document(String header) {
        // Imagine this takes 2 seconds to load from a Database
        this.header = header;
    }

    public void addLine(String text) {
        content.add(text);
    }

    @Override
    public Document clone() throws CloneNotSupportedException {
        // Default clone() is a "Shallow Copy"
        Document copy = (Document) super.clone();
        
        // Manual "Deep Copy" for the list to ensure the clone 
        // doesn't share the same list reference as the original
        copy.content = new ArrayList<>(this.content);
        return copy;
    }

    @Override
    public String toString() {
        return "Document [Header=" + header + ", Content=" + content + "]";
    }
}
```
**Using the Prototype:**

Java

```
public class Main {
    public static void main(String[] args) throws CloneNotSupportedException {
        // Create the original (expensive)
        Document original = new Document("Standard Report");
        original.addLine("Intro: This is a template.");

        // Clone it (cheap and fast)
        Document quarterlyReport = original.clone();
        quarterlyReport.addLine("Update: Q1 looks great!");

        System.out.println(original);
        System.out.println(quarterlyReport);
    }
}
```
---

### Shallow Copy vs. Deep Copy
This is the most critical technical detail when implementing the Prototype pattern:

| **Feature** | **Shallow Copy** | **Deep Copy** |
| ----- | ----- | ----- |
| **Primitive Fields** | Copied (int, double, etc.) | Copied |
| **Object References** | <p>Shares the </p><p>**same reference**</p><p>.</p> | <p>Creates a </p><p>**new instance**</p><p> of the object.</p> |
| **Risk** | Changing a list in the clone changes it in the original. | Clone is 100% independent. |
| **Complexity** | Simple (`super.clone()`). | Requires manual code for every nested object. |
---

### Pros and Cons
- **Pros:**
    - Reduces the need for repetitive initialization code.
    - Allows you to add/remove products at runtime by simply adding new prototypes to a registry.
    - Hides the complexity of object creation from the client.

- **Cons:**
    - **Circular References:** Cloning objects with circular references (Object A points to B, and B points to A) is notoriously difficult.
    - **Implementation Overhead:** Every class in the hierarchy must implement the cloning logic, which can be tedious if you have many nested objects.

**Yes**, the example I provided is a **Deep Clone**.

To understand why, we have to look at how Java handles memory. In the `clone()` method of that `Document` class, there were two distinct steps:

### 1. The Shallow Part (`super.clone()`)
When we call `super.clone()`, Java creates a new instance of the object and copies all the field values.

- For the `header`  (a `String` ), this is fine because Strings are immutable in Java.
- However, for the `content`  (a `List` ), it only copies the **reference** (the memory address).
If we stopped there, both the original and the clone would be pointing to the **exact same list** in memory. If you added a line to the clone, it would magically appear in the original too. This is a **Shallow Clone**.

---

### 2. The Deep Part (`new ArrayList<>(...)`)
This is the specific line that turned it into a **Deep Clone**:

Java

```
copy.content = new ArrayList<>(this.content);
```
By explicitly creating a `new ArrayList`, we told Java: _"Don't just point to the old list. Create a brand new list in a different spot in memory and fill it with the same items."_

Now, the two objects are completely independent. If you modify the list in one, the other remains untouched.

---

### How to tell the difference at a glance:
| **Feature** | **Shallow Clone** | **Deep Clone (The Example)** |
| ----- | ----- | ----- |
| **Logic** | `return (Document) super.clone();`  | <p>`Document c = (Document) super.clone();` </p><p>`c.field = new Field(this.field);` </p> |
| **Memory** | Both objects share the same internal objects/collections. | Both objects have their own private copies of internal objects. |
| **Independence** | <p>Changing a list in the clone </p><p>**affects**</p><p> the original.</p> | <p>Changing a list in the clone </p><p>**does not affect**</p><p> the original.</p> |
### A Quick Warning
In a "Real World" enterprise app, manual deep cloning can get very messy if your objects are deeply nested (e.g., a `Document` contains a `Page`, which contains a `Paragraph`, which contains a `Style`).

In those cases, developers often avoid `Cloneable` and instead use:

1. **Copy Constructors**: `public Document(Document other) { ... }` 
2. **Serialization**: Converting the object to JSON/XML and back to a new object.
3. **ModelMapper/MapStruct**: Libraries that handle this deep copying automatically.




<!--- Eraser file: https://app.eraser.io/workspace/9DAaxO05AuRKxTiofNbK --->