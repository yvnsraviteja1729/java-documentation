<p><a target="_blank" href="https://app.eraser.io/workspace/FAaLrVBVA1hGv4d9mFsn" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>



# 1. What Is a Custom Annotation?
An **annotation** is **metadata** added to:

- Classes
- Methods
- Fields
- Parameters
It does **not execute code by itself**.
➡ Someone (compiler, framework, AOP, reflection) **reads and acts on it**.

---

# 2. Basic Syntax of a Custom Annotation
```java
public @interface MyAnnotation {
}
```
But this alone is **not useful** yet.

---

# 3. Mandatory Annotation Meta-Annotations
Java provides **meta-annotations** to control behavior.

### 3.1 `@Target` – Where annotation can be used
```java
@Target(ElementType.METHOD)
```
Common targets:

| ElementType | Usage |
| ----- | ----- |
| TYPE | Class, Interface |
| METHOD | Method |
| FIELD | Variable |
| PARAMETER | Method parameter |
| CONSTRUCTOR | Constructor |
| ANNOTATION_TYPE | Another annotation |
---

### 3.2 `@Retention` – How long annotation is available
```java
@Retention(RetentionPolicy.RUNTIME)
```
| Policy | Available |
| ----- | ----- |
| SOURCE | Compile time only |
| CLASS | <p>In </p><p> file</p> |
| RUNTIME | Available via reflection |
👉 **Spring AOP requires **`**RUNTIME**` 

---

### 3.3 Basic Custom Annotation Example
```java
import java.lang.annotation.*;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface LogExecutionTime {
}
```
---

# 4. Using the Annotation
```java
@Service
public class OrderService {

    @LogExecutionTime
    public void placeOrder() {
        // business logic
    }
}
```
---

# 5. Reading Custom Annotation Using Spring AOP (Real-World)
### Aspect that reacts to annotation
```java
@Aspect
@Component
public class LoggingAspect {

    @Around("@annotation(LogExecutionTime)")
    public Object logTime(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();

        Object result = pjp.proceed();

        long end = System.currentTimeMillis();
        System.out.println(
            pjp.getSignature() + " took " + (end - start) + " ms"
        );

        return result;
    }
}
```
➡ Annotation + AOP = **powerful combination**

---

# 6. Annotation with Parameters
### Define annotation
```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Retry {

    int attempts() default 3;
    long delay() default 1000; // ms
}
```
---

### Use annotation
```java
@Retry(attempts = 5, delay = 2000)
public void callExternalService() {
}
```
---

### Access annotation values in AOP
```java
@Around("@annotation(retry)")
public Object retryLogic(
        ProceedingJoinPoint pjp,
        Retry retry) throws Throwable {

    int attempts = retry.attempts();
    long delay = retry.delay();

    for (int i = 1; i <= attempts; i++) {
        try {
            return pjp.proceed();
        } catch (Exception ex) {
            if (i == attempts) throw ex;
            Thread.sleep(delay);
        }
    }
    return null;
}
```
---

# 7. Annotation on Class Level
### Define
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface Secured {
    String role();
}
```
---

### Use
```java
@Secured(role = "ADMIN")
@Service
public class AdminService {
}
```
---

### Aspect
```java
@Before("@within(secured)")
public void checkRole(Secured secured) {
    System.out.println("Required role: " + secured.role());
}
```
---

# 8. Annotation with Multiple Targets
```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface Auditable {
}
```
---

# 9. Reading Annotation Without AOP (Reflection)
```java
Method method = OrderService.class
        .getMethod("placeOrder");

if (method.isAnnotationPresent(LogExecutionTime.class)) {
    System.out.println("Annotation present");
}
```
---

# 10. Meta-Annotations (Advanced)
### 10.1 `@Inherited` 
Allows subclasses to inherit annotation

```java
@Inherited
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface InheritedConfig {
}
```
---

### 10.2 `@Repeatable` 
Allow multiple annotations

```java
@Repeatable(Roles.class)
public @interface Role {
    String value();
}

public @interface Roles {
    Role[] value();
}
```
---

# 11. Real-World Spring Examples Using Custom Annotations
| Annotation | Purpose |
| ----- | ----- |
|  | Transaction management |
|  | Caching |
|  | Security |
|  | Async execution |
➡ All are implemented using **AOP + reflection**

---

# 12. Common Mistakes (Very Important)
❌ Forgetting `@Retention(RUNTIME)`
❌ Expecting annotation to execute code by itself
❌ Using annotation without reader (Aspect / Processor)
❌ Putting business logic inside annotation

---

# 13. Best Practices
✔ Keep annotations **small and declarative**
✔ Use meaningful names
✔ Combine annotation + AOP
✔ Document annotation parameters
✔ Avoid overengineering

---

# Mental Model (One Line)
> **Annotation = Marker**
**AOP / Reflection = Brain that reacts to the marker**

---





<!--- Eraser file: https://app.eraser.io/workspace/FAaLrVBVA1hGv4d9mFsn --->