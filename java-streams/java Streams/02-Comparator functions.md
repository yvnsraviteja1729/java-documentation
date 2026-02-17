<p><a target="_blank" href="https://app.eraser.io/workspace/c5LI6ut0Z9gBVKKj0Fno" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>

# Comparator functions
Comparator in Java

A `Comparator<T>` is a functional interface (`compare(T a, T b)`) used to define custom ordering.

In Java 8+, you often use **static helper methods** and **lambdas** to create them.

---

## 1. Natural and Reverse Ordering
```java
Comparator.naturalOrder();   // ascending (uses compareTo)
Comparator.reverseOrder();   // descending (reverse compareTo)
```
---

## 2. Comparing by a Key (Extracting field or property)
### For objects
```java
Comparator.comparing(Person::getName);      // sort by name
Comparator.comparing(Person::getAge);       // sort by age
Comparator.comparing(Person::getSalary);    // sort by salary
```
### For primitives
```java
Comparator.comparingInt(String::length);         // by int key
Comparator.comparingLong(Employee::getId);       // by long key
Comparator.comparingDouble(Product::getPrice);   // by double key
```
---

## 3. Reverse and Null Handling
```java
Comparator.comparing(Person::getAge).reversed();            // reverse order
Comparator.nullsFirst(Comparator.naturalOrder());           // nulls come first
Comparator.nullsLast(Comparator.comparing(String::length)); // nulls come last
```
---

## 4. Multiple-Level (Tie-Breaking)
```java
Comparator.comparing(Person::getLastName)
          .thenComparing(Person::getFirstName);   // last name, then first name
Comparator.comparingInt(String::length)
          .thenComparing(Comparator.naturalOrder()); // by length, then alphabetical
```
---

## 5. Custom Lambda Comparators
```java
Comparator<String> byLength = (a, b) -> Integer.compare(a.length(), b.length());
Comparator<Integer> reverse = (a, b) -> b - a;  // reverse numeric
```
---

## 6. Useful Ready-to-Use Comparators
- `Comparator.naturalOrder()`  → ascending (like `a.compareTo(b)` ).
- `Comparator.reverseOrder()`  → descending.
- `Comparator.comparing(...)`  → key extractor (with option to chain).
- `Comparator.comparingInt` , `comparingLong` , `comparingDouble`  → primitive optimized.
- `Comparator.nullsFirst(...)` , `Comparator.nullsLast(...)`  → handle `null`  values.
---

# 🔹 Examples in Action
### Sort strings by length, then alphabetically
```java
list.stream()
.sorted(Comparator.comparingInt(String::length)
                  .thenComparing(Comparator.naturalOrder()))
.forEach(System.out::println);
```
### Sort employees by salary (descending), then name
```java
list.stream()
.sorted(Comparator.comparingDouble(Employee::getSalary).reversed()
                  .thenComparing(Employee::getName))
.forEach(System.out::println);
```
---

⚡ In short:

- Use `**comparing**`  when sorting by a property.
- Add `**.reversed()**`  for descending.
- Add `**.thenComparing()**`  for tie-breaking.
- Wrap with `**nullsFirst/nullsLast**`  when nullable.
---





<!--- Eraser file: https://app.eraser.io/workspace/c5LI6ut0Z9gBVKKj0Fno --->