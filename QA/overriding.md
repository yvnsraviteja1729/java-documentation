You can change the access modifier when overriding a method in Java, but **only if you make it more accessible (less restrictive)**. You cannot make it more restrictive.

The core rule in Java is that an overriding method cannot assign weaker access privileges than the method had in the parent class.

---

## The Visibility Hierarchy

To understand what is allowed, it helps to look at the access levels from **most restrictive** to **least restrictive**:

$$\text{private} \longrightarrow \text{package-private (default)} \longrightarrow \text{protected} \longrightarrow \text{public}$$

---

## What is Allowed vs. What is Forbidden

### 1. Allowed (Expanding Visibility)

You can change the modifier to any level that is to the *right* of the original modifier in the hierarchy.

* **If parent is `protected`:** The child method can be `protected` or `public`.
* **If parent is package-private (default):** The child method can be default, `protected`, or `public`.

```java
class Parent {
    protected void display() {
        System.out.println("Parent");
    }
}

class Child extends Parent {
    @Override
    public void display() { // Allowed! 'public' is less restrictive than 'protected'
        System.out.println("Child");
    }
}

```

### 2. Forbidden (Restricting Visibility)

You cannot change the modifier to anything to the *left* of the original modifier. Doing so will result in a **compile-time error**: *"cannot reduce the visibility of the inherited method"*.

* **If parent is `public`:** The child method **must** be `public`. It cannot be `protected`, default, or `private`.
* **If parent is `protected`:** The child method cannot be default or `private`.

```java
class Parent {
    public void show() {
        System.out.println("Parent");
    }
}

class Child extends Parent {
    @Override
    protected void show() { // Compile-time Error! Cannot reduce visibility from public to protected
        System.out.println("Child");
    }
}

```

---

## What about `private` methods?

You **cannot** override a `private` method at all.

Because `private` methods are completely hidden from child classes, a method with the same signature in the child class is treated as a brand-new, independent method. It is a case of **method hiding**, not method overriding. Therefore, the `@Override` annotation will throw an error if you try to use it.

---

## Why does Java enforce this? (The Liskov Substitution Principle)

This rule exists to uphold polymorphism. If a piece of code holds a reference to a `Parent` class object, it expects to have access to all of the `Parent`'s public API.

If a `Child` object could sneak in and hide a public method by making it `private`, the code relying on the `Parent` reference would suddenly crash with an access error at runtime. Java prevents this loophole at compile time.