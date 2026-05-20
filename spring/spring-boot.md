## 1️⃣ How does Spring Boot Auto-Configuration work internally?

**Auto-Configuration** automatically configures Spring beans based on:
- Classpath dependencies (jars present)
- Existing beans
- Properties set in `application.properties`/`yml`

### 🔹 Internal Flow

1. `@SpringBootApplication` includes `@EnableAutoConfiguration`.
2. `@EnableAutoConfiguration` imports **`AutoConfigurationImportSelector`**.
3. This selector scans the classpath for:
   - **Spring Boot 2.x:** `META-INF/spring.factories`
   - **Spring Boot 2.7+/3.x:** `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`
4. It loads a list of **`@Configuration` classes** (e.g., `DataSourceAutoConfiguration`, `WebMvcAutoConfiguration`).
5. Each auto-config class uses **`@Conditional…`** annotations:
   - `@ConditionalOnClass` → only if class exists in classpath
   - `@ConditionalOnMissingBean` → only if user hasn't defined one
   - `@ConditionalOnProperty` → only if property set
6. If conditions match → bean is registered into the `ApplicationContext`.

### Example
```java
@Configuration
@ConditionalOnClass(DataSource.class)
@ConditionalOnMissingBean(DataSource.class)
public class DataSourceAutoConfiguration {
    @Bean
    public DataSource dataSource() { ... }
}
```

➡ Add `spring-boot-starter-data-jpa` → `DataSource`, `EntityManagerFactory`, `TransactionManager` auto-configured.

---

## 2️⃣ What exactly happens when `@SpringBootApplication` is used?

`@SpringBootApplication` is a **meta-annotation**:

```java
@SpringBootApplication
= @SpringBootConfiguration   // (variant of @Configuration)
+ @EnableAutoConfiguration   // triggers auto-config
+ @ComponentScan             // scans current package & sub-packages
```

### Internal Steps
1. Marks the class as a **Spring configuration class**.
2. Enables **auto-configuration** via `AutoConfigurationImportSelector`.
3. Triggers **component scanning** from the class's package downward.
4. When `SpringApplication.run()` is invoked, it bootstraps the entire application.

---

## 3️⃣ How does Spring Boot decide which beans to auto-configure?

It uses **conditional evaluation**:

| Annotation | Trigger |
|-----------|---------|
| `@ConditionalOnClass` | Class is present on classpath |
| `@ConditionalOnMissingClass` | Class is absent |
| `@ConditionalOnBean` | Specific bean is registered |
| `@ConditionalOnMissingBean` | Bean not already defined |
| `@ConditionalOnProperty` | Property exists / matches value |
| `@ConditionalOnWebApplication` | App is web-based |
| `@ConditionalOnExpression` | SpEL evaluates true |

➡ The **`ConditionEvaluator`** processes these at startup to decide which beans to create.

You can debug this with:
```properties
debug=true
```
This prints **Positive/Negative matches** report.

---

## 4️⃣ Role of `spring.factories` / `AutoConfiguration.imports`

These files act as a **registry of auto-configuration classes**.

### Spring Boot 2.x — `META-INF/spring.factories`
```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.example.MyAutoConfiguration,\
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

### Spring Boot 2.7+ / 3.x — `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`
```
com.example.MyAutoConfiguration
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

➡ Spring Boot scans these files in **all JARs** and loads them as candidate auto-configurations.
➡ Helps **third-party libraries** plug into Boot's auto-config system.

---

## 5️⃣ Difference between `@ComponentScan` and `@EnableAutoConfiguration`

| Feature | `@ComponentScan` | `@EnableAutoConfiguration` |
|---------|------------------|---------------------------|
| Purpose | Scans your **own** package for components | Loads **Spring Boot's** pre-built config classes |
| What it finds | `@Component`, `@Service`, `@Controller`, `@Repository` | `@Configuration` classes from `AutoConfiguration.imports` |
| Scope | Your code | Library/framework |
| Conditional? | No | Yes (`@Conditional…`) |

➡ `@ComponentScan` = **your beans**
➡ `@EnableAutoConfiguration` = **framework beans**

---

## 6️⃣ How does Spring Boot's Embedded Tomcat start internally?

1. `spring-boot-starter-web` adds Tomcat (`tomcat-embed-core`) jar.
2. Auto-config detects `Tomcat.class` on classpath → triggers `ServletWebServerFactoryAutoConfiguration`.
3. A **`TomcatServletWebServerFactory`** bean is created.
4. When `ApplicationContext` is refreshed:
   - `ServletWebServerApplicationContext.onRefresh()` is called.
   - It invokes `createWebServer()`.
5. Tomcat is **programmatically started** (not deployed):
   ```java
   Tomcat tomcat = new Tomcat();
   tomcat.setPort(port);
   tomcat.start();
   ```
6. `DispatcherServlet` is registered on Tomcat as the root servlet.
7. Tomcat listens for requests → routes them to Spring MVC.

➡ Whole server runs **inside the JVM process** — no external WAR deployment.

---

## 7️⃣ How does Spring Boot create and manage beans internally?

Spring Boot delegates to the **Spring IoC container** (`ApplicationContext`).

### Internal Steps
1. **BeanDefinition** objects are created from:
   - `@Component`, `@Service`, etc. (via component scan)
   - `@Bean` methods (via `@Configuration`)
2. Stored in **`BeanDefinitionRegistry`**.
3. During `refresh()`:
   - **BeanFactoryPostProcessors** run (e.g., property placeholder resolution).
   - **BeanPostProcessors** registered.
   - Beans are **instantiated** (constructor injection).
   - **Dependencies injected** (field/setter).
   - `@PostConstruct` invoked.
   - Beans ready for use.
4. On shutdown → `@PreDestroy` called → beans destroyed.

---

## 8️⃣ Complete Spring Boot Application Startup Lifecycle

```
main() → SpringApplication.run()
        │
        ▼
1. Create SpringApplication instance
2. Detect application type (Servlet / Reactive / None)
3. Setup default Initializers & Listeners (from spring.factories)
4. Prepare Environment (load application.properties / profiles / env vars)
5. Print Banner
6. Create ApplicationContext (AnnotationConfigServletWebServerApplicationContext)
7. Prepare context (apply initializers, register sources)
8. refresh() → core lifecycle:
       a. Load BeanDefinitions
       b. Invoke BeanFactoryPostProcessors
       c. Register BeanPostProcessors
       d. Instantiate singletons
       e. Trigger Auto-configurations
       f. Start embedded server (Tomcat/Netty)
9. Call ApplicationRunner & CommandLineRunner
10. Application Ready Event published
```

---

## 9️⃣ How do Profiles work internally in Spring Boot?

Profiles allow **environment-specific configuration**.

### Internal Mechanism
- `Environment` object holds **active profiles** (`spring.profiles.active`).
- Spring loads profile-specific property files:
  - `application-dev.properties`
  - `application-prod.properties`
- Beans annotated with `@Profile("dev")` are **conditionally registered**.

### Example
```java
@Profile("prod")
@Bean
DataSource prodDataSource() { ... }
```

Activate:
```bash
--spring.profiles.active=prod
```

➡ The `ProfileCondition` checks active profiles before bean registration.

---

## 🔟 How does Externalized Configuration work?

Spring Boot can load properties from many sources, allowing config to live **outside the JAR**.

### Sources (Highest → Lowest Priority)
1. Command-line args (`--server.port=9090`)
2. `SPRING_APPLICATION_JSON` env var
3. OS environment variables
4. Java system properties (`-Dkey=value`)
5. `application-{profile}.properties/yml`
6. `application.properties/yml`
7. `@PropertySource` annotated configs
8. Default values

➡ Managed via **`PropertySource` abstraction** inside `Environment`.

---

## 1️⃣1️⃣ Order of Property Resolution in Spring Boot

Spring Boot uses a layered **`PropertySources`** list. Top wins.

```
1. DevTools properties (~/.spring-boot-devtools.properties)
2. Test properties (@TestPropertySource)
3. Command-line args
4. SPRING_APPLICATION_JSON
5. ServletConfig / ServletContext params
6. JNDI attributes
7. Java system properties
8. OS env vars
9. Profile-specific application-{profile}.yml
10. application.yml / application.properties
11. @PropertySource on @Configuration
12. SpringApplication.setDefaultProperties()
```

---

## 1️⃣2️⃣ How does Spring Boot Actuator work internally?

Actuator exposes **management endpoints** like `/health`, `/metrics`, `/env`.

### Internal Working
1. `spring-boot-starter-actuator` triggers `EndpointAutoConfiguration`.
2. Each endpoint = bean annotated with `@Endpoint`, `@WebEndpoint`, etc.
3. Endpoints are exposed via:
   - **HTTP** (Web)
   - **JMX**
4. `WebMvcEndpointHandlerMapping` maps `/actuator/*` URLs to endpoints.
5. Uses **Micrometer** internally for metrics collection.
6. Endpoints can be enabled/disabled:
   ```properties
   management.endpoints.web.exposure.include=health,info,metrics
   ```

---

## 1️⃣3️⃣ Custom Health Indicator Example

```java
@Component
public class DatabaseHealthIndicator implements HealthIndicator {

    @Override
    public Health health() {
        boolean dbUp = checkDatabase();
        if (dbUp) return Health.up().withDetail("DB", "Reachable").build();
        return Health.down().withDetail("DB", "Unreachable").build();
    }

    private boolean checkDatabase() { /* ping DB */ return true; }
}
```

➡ Spring auto-detects all `HealthIndicator` beans and includes them in `/actuator/health`.

---

## 1️⃣4️⃣ How does Spring Boot DevTools work?

Adds **developer productivity features**:

### Internal Mechanism
- Uses **two classloaders**:
  - **Base classloader** → unchanged libraries (jars)
  - **Restart classloader** → your application classes
- When code changes:
  - DevTools detects changes in classpath.
  - Throws away the restart classloader.
  - Reloads only your app classes → **fast restart**.
- Includes **LiveReload server** for browser auto-refresh.
- Disables template caching for hot reload.

```xml
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-devtools</artifactId>
   <scope>runtime</scope>
</dependency>
```

---

## 1️⃣5️⃣ What happens internally when `@ConfigurationProperties` is used?

```java
@ConfigurationProperties(prefix = "app")
public class AppProps {
    private String name;
    private int timeout;
}
```

### Internal Steps
1. Spring registers a `ConfigurationPropertiesBindingPostProcessor`.
2. At bean init:
   - Reads `Environment` properties matching prefix.
   - Uses **`Binder` API** to bind values via setters / constructors.
   - Performs **type conversion** and **validation** (with `@Validated`).
3. Bean populated with values.

➡ More robust than `@Value`: supports **relaxed binding** (`app.name` ↔ `APP_NAME`), nested properties, lists, validation.

---

## 1️⃣6️⃣ `@Value` vs `@ConfigurationProperties` Internally

| Feature | `@Value` | `@ConfigurationProperties` |
|---------|----------|----------------------------|
| Binding | Single property, SpEL | Bulk binding via Binder API |
| Type-safe | ❌ | ✅ |
| Relaxed binding | ❌ | ✅ |
| Validation (`@Valid`) | ❌ | ✅ |
| Nested objects | ❌ | ✅ |
| Internal mechanism | `BeanPostProcessor` resolves `${}` at injection | `ConfigurationPropertiesBindingPostProcessor` binds whole class |

➡ Use `@ConfigurationProperties` for **structured configs**, `@Value` for **one-off injections**.

---

## 1️⃣7️⃣ Global Exception Handling Internals

Done using `@ControllerAdvice` + `@ExceptionHandler`.

### Internal Working
1. `ExceptionHandlerExceptionResolver` (one of Spring's `HandlerExceptionResolver`s) detects unhandled exceptions.
2. It looks for matching `@ExceptionHandler` methods in:
   - Controller class
   - `@ControllerAdvice` beans
3. Invokes the matching method, returning a `ResponseEntity` or view.

### Example
```java
@RestControllerAdvice
public class GlobalHandler {
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<String> notFound(Exception e) {
        return ResponseEntity.status(404).body(e.getMessage());
    }
}
```

---

## 1️⃣8️⃣ What happens internally when `@RestController` returns a response?

1. Request hits **`DispatcherServlet`**.
2. `HandlerMapping` finds controller method.
3. `HandlerAdapter` invokes the method.
4. Returned object goes through **`HttpMessageConverter`** (e.g., `MappingJackson2HttpMessageConverter`).
5. Converter serializes object → JSON → written to `HttpServletResponse`.
6. `Content-Type: application/json` set automatically.

➡ `@RestController` = `@Controller` + `@ResponseBody` → bypasses view resolution.

---

## 1️⃣9️⃣ How does Spring Boot perform request-to-object mapping?

When a request body or parameter comes in:

### For `@RequestBody`
1. `RequestMappingHandlerAdapter` invokes **`HandlerMethodArgumentResolver`**.
2. `RequestResponseBodyMethodProcessor` handles `@RequestBody`.
3. Uses `HttpMessageConverter` (Jackson) to convert JSON → Java POJO.
4. Validates with `@Valid` (via `Validator` API).

### For `@RequestParam` / `@PathVariable`
- Resolved by `RequestParamMethodArgumentResolver` / `PathVariableMethodArgumentResolver`.
- Spring uses **`ConversionService`** to convert string → target type.

---

## 2️⃣0️⃣ How does Spring Boot manage connection pooling automatically?

1. `DataSourceAutoConfiguration` detects `spring.datasource.*` properties.
2. It checks classpath for available pool implementations:
   - **HikariCP** (default in Spring Boot 2+)
   - Tomcat JDBC
   - Commons DBCP2
3. Auto-creates a `DataSource` bean using `DataSourceBuilder`.
4. Connection pool configured with sensible defaults + overridable properties:
   ```properties
   spring.datasource.hikari.maximum-pool-size=20
   spring.datasource.hikari.idle-timeout=30000
   ```

---

## 2️⃣1️⃣ Why is HikariCP the default Connection Pool?

✅ **Fastest** JDBC connection pool (benchmarked).
✅ **Lightweight** (~130KB jar).
✅ **Concurrent** — uses lock-free algorithms.
✅ **Reliable** — handles connection leaks well.
✅ **Default since Spring Boot 2.0**.

➡ Spring Boot auto-detects HikariCP in classpath (`com.zaxxer.hikari.HikariDataSource`) and prefers it over others.

---

## 2️⃣2️⃣ How does Spring Boot Caching work internally?

1. `@EnableCaching` registers a **`CachingConfigurer`** and post-processors.
2. **`ProxyCachingConfiguration`** creates AOP proxies around beans with `@Cacheable`, `@CachePut`, `@CacheEvict`.
3. When method is called:
   - **`CacheInterceptor`** intercepts it.
   - Computes cache key (via `KeyGenerator`).
   - Looks up cache via `CacheManager.getCache(name)`.
   - If hit → returns cached value (method NOT executed).
   - If miss → executes method → stores result → returns.
4. Default `CacheManager`:
   - `ConcurrentMapCacheManager` (simple)
   - Auto-replaced if Redis/Caffeine/EhCache present.

### Example
```java
@Cacheable(value = "users", key = "#id")
public User getUser(Long id) { ... }
```

---

## 2️⃣3️⃣ What happens internally when `@Async` is enabled?

1. `@EnableAsync` registers `AsyncAnnotationBeanPostProcessor`.
2. This wraps beans containing `@Async` methods in **AOP proxies**.
3. When the method is called:
   - The proxy submits the call to a **`TaskExecutor`** (`SimpleAsyncTaskExecutor` by default).
   - Returns immediately:
     - `void` → fire & forget
     - `Future` / `CompletableFuture` → caller can await.
4. Original caller thread continues without blocking.

### Example
```java
@EnableAsync
@SpringBootApplication
public class App {}

@Async
public CompletableFuture<String> fetchData() {
    return CompletableFuture.completedFuture("Done");
}
```

⚠ Limitation: Async only works for **public methods called from outside the class** (AOP proxy limitation).

---

## 2️⃣4️⃣ How does Spring Boot support Graceful Shutdown?

Since **Spring Boot 2.3**, supports graceful shutdown by completing in-flight requests before stopping.

### Enable
```properties
server.shutdown=graceful
spring.lifecycle.timeout-per-shutdown-phase=30s
```

### Internal Mechanism
1. On `SIGTERM`, `ApplicationContext.close()` is triggered.
2. Embedded server stops **accepting new requests**.
3. In-flight requests are given the shutdown timeout to complete.
4. Then `@PreDestroy` lifecycle invoked → resources cleaned (DB pools, threads).
5. Finally JVM exits.

➡ Crucial in **Kubernetes** to handle pod termination properly.

---

## 2️⃣5️⃣ How to Optimize Spring Boot Startup Time in Production?

### ⚡ Application-level Tips
1. **Lazy initialization**
   ```properties
   spring.main.lazy-initialization=true
   ```
2. **Exclude unused auto-configurations**
   ```java
   @SpringBootApplication(exclude = {SecurityAutoConfiguration.class})
   ```
3. **Disable JMX** if not needed
   ```properties
   spring.jmx.enabled=false
   ```
4. **Trim classpath** (remove unused starters).
5. **Use specific component scanning**
   ```java
   @ComponentScan(basePackages = "com.example.app")
   ```

### ⚡ JVM-level Tips
6. Use **JVM tiered compilation**:
   ```
   -XX:TieredStopAtLevel=1
   ```
7. Use **CDS (Class Data Sharing)** for faster class loading.
8. Use **GraalVM Native Image** (Spring Boot 3) → boot in milliseconds.

### ⚡ Build-Level Tips
9. **Spring Native / AOT processing** (`spring-boot-starter-parent` 3.x).
10. Use **Layered Jars** for Docker:
   ```bash
   java -Djarmode=layertools -jar app.jar extract
   ```
11. Profile with `--debug` and **Actuator startup endpoint**:
   ```
   /actuator/startup
   ```

---

## 🧠 Quick Recap Cheatsheet

| Concept | Internal Mechanism |
|---------|--------------------|
| Auto-Configuration | `@Conditional…` based bean registration |
| `@SpringBootApplication` | Combines 3 annotations |
| Embedded Tomcat | Started via `ServletWebServerFactory` |
| Profiles | Driven by `Environment` + `ProfileCondition` |
| Actuator | Endpoint beans + Micrometer |
| DevTools | Dual classloader + restart |
| `@ConfigurationProperties` | Binder API + post-processor |
| `@Async` | AOP proxy + TaskExecutor |
| Caching | AOP + CacheManager + KeyGenerator |
| HikariCP | Default, fastest pool |
| Graceful shutdown | `server.shutdown=graceful` |
| Startup optimization | Lazy init, exclude configs, AOT, GraalVM |

---
