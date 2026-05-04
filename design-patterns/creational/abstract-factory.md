<p><a target="_blank" href="https://app.eraser.io/workspace/5HkT24mcXkyJblHlueFf" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>

The **Abstract Factory Pattern** is often called a "Factory of Factories." While the Factory Method pattern creates **one** type of product, the Abstract Factory creates a **family** of related products without you having to specify their concrete classes.

Think of it as a higher level of abstraction. If the Factory Method is a single assembly line, the Abstract Factory is the entire factory building that coordinates multiple assembly lines to ensure everything matches.

---

### Factory Method vs. Abstract Factory
| **Feature** | **Factory Method** | **Abstract Factory** |
| ----- | ----- | ----- |
| **Focus** | Creates one product. | Creates families of related products. |
| **Mechanism** | Uses inheritance (subclasses override a method). | Uses object composition (a factory object is passed around). |
| **Complexity** | Simple; good for single items. | Higher; ensures products are compatible with each other. |
---

### Real-World Example: Cross-Platform UI Toolkit
Imagine you are building a UI library that needs to work on both **Windows** and **Mac**. You have different components like **Buttons** and **Checkboxes**.

A Windows Button should never be paired with a Mac Checkbox—they must match the OS "theme."

#### 1. The Product Interfaces (The "Family")
Java

```
public interface Button {
    void paint();
}

public interface Checkbox {
    void paint();
}
```
#### 2. The Abstract Factory Interface
Java

```
public interface GUIFactory {
    Button createButton();
    Checkbox createCheckbox();
}
```
#### 3. Concrete Factories (The "Themes")
Java

```
// Windows Factory ensures everything is Windows-styled
public class WindowsFactory implements GUIFactory {
    @Override
    public Button createButton() { return new WindowsButton(); }
    
    @Override
    public Checkbox createCheckbox() { return new WindowsCheckbox(); }
}

// Mac Factory ensures everything is Mac-styled
public class MacFactory implements GUIFactory {
    @Override
    public Button createButton() { return new MacButton(); }
    
    @Override
    public Checkbox createCheckbox() { return new MacCheckbox(); }
}
```
#### 4. The Client Code
The client doesn't care if it's using Windows or Mac; it just knows it has a `GUIFactory`.

Java

```
public class Application {
    private Button button;
    private Checkbox checkbox;

    public Application(GUIFactory factory) {
        // The factory handles the "matching" logic
        button = factory.createButton();
        checkbox = factory.createCheckbox();
    }

    public void paint() {
        button.paint();
        checkbox.paint();
    }
}
```
---

### Why use this?
- **Consistency**: You guarantee that products from the same factory are compatible. You won't accidentally mix "Waterproof" parts with "Electronic" parts that aren't meant to touch water.
- **Single Responsibility**: You extract the product creation code into one place.
- **Open/Closed Principle**: You can introduce new variants of products (like a "Linux" theme) without breaking the existing client code.
> **Pro-Tip:** In modern Java/Spring development, you often see this pattern when dealing with different database dialects (SQL Server vs. Oracle) or different cloud providers (AWS vs. Azure), where each provider requires a "family" of specific connection and storage implementations.





<!--- Eraser file: https://app.eraser.io/workspace/5HkT24mcXkyJblHlueFf --->