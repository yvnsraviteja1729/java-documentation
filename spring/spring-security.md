---

# 🔐 PART 1 — Spring Security, CORS & CSRF

---

## 1️⃣ What is CORS and Why is it Required?

**CORS (Cross-Origin Resource Sharing)** is a **browser security mechanism** that controls how web pages from one origin can request resources from another origin.

### 🔹 What's an "Origin"?
`protocol + domain + port` → e.g., `https://app.example.com:8080`

### 🔹 The Problem
By default, browsers enforce the **Same-Origin Policy (SOP)** — JavaScript from `frontend.com` **cannot** call APIs at `api.backend.com` directly.

### 🔹 The Solution
CORS allows the **server to explicitly tell the browser** which origins, methods, and headers are allowed.

### 🔹 How It Works
1. Browser sends an HTTP request with `Origin: https://frontend.com`.
2. For non-simple requests, sends a **preflight `OPTIONS`** request first.
3. Server responds with headers:
   ```
   Access-Control-Allow-Origin: https://frontend.com
   Access-Control-Allow-Methods: GET, POST, PUT
   Access-Control-Allow-Headers: Authorization, Content-Type
   ```
4. If allowed → browser sends actual request.

✅ Required for **decoupled frontend-backend apps** (React/Angular ↔ Spring Boot APIs).

---

## 2️⃣ How to Configure CORS in Spring Boot?

### 🔹 Option 1 — Global CORS (Recommended)
```java
@Configuration
public class CorsConfig {
    @Bean
    public WebMvcConfigurer corsConfigurer() {
        return new WebMvcConfigurer() {
            @Override
            public void addCorsMappings(CorsRegistry registry) {
                registry.addMapping("/api/**")
                        .allowedOrigins("https://frontend.com")
                        .allowedMethods("GET", "POST", "PUT", "DELETE")
                        .allowedHeaders("*")
                        .allowCredentials(true);
            }
        };
    }
}
```

### 🔹 Option 2 — Controller/Method-Level
```java
@CrossOrigin(origins = "https://frontend.com")
@RestController
public class UserController { ... }
```

### 🔹 Option 3 — With Spring Security
```java
@Bean
SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http.cors(Customizer.withDefaults())
        .csrf(csrf -> csrf.disable())
        .authorizeHttpRequests(auth -> auth.anyRequest().authenticated());
    return http.build();
}

@Bean
CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("https://frontend.com"));
    config.setAllowedMethods(List.of("GET", "POST"));
    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", config);
    return source;
}
```

---

## 3️⃣ What is a CSRF Attack?

**CSRF (Cross-Site Request Forgery)** = an attacker tricks a **logged-in user's browser** into making **unwanted requests** to a trusted site.

### 🔹 Example Scenario
1. User logs into `bank.com` → session cookie stored in browser.
2. User visits `malicious.com` containing:
   ```html
   <img src="https://bank.com/transfer?to=hacker&amount=10000">
   ```
3. Browser sends request with **session cookie attached** automatically.
4. Bank thinks it's a legitimate request → money transferred. 💸

### 🔹 Why It Works
- Cookies are sent automatically with cross-site requests.
- Server can't tell if request came from the legitimate user or an attacker page.

---

## 4️⃣ How does Spring Security Prevent CSRF?

Spring Security uses the **Synchronizer Token Pattern**:

### 🔹 Mechanism
1. Server generates a **CSRF token** per session.
2. Token is embedded in HTML form / sent via header.
3. On every state-changing request (POST/PUT/DELETE), the client must include the token.
4. Server compares it with the one stored in session.
5. ❌ Mismatch → request rejected with `403`.

### 🔹 Default Behavior
CSRF protection is **enabled by default** in Spring Security.

### 🔹 Example
```html
<input type="hidden" name="_csrf" value="${_csrf.token}"/>
```

Or in headers:
```
X-CSRF-TOKEN: <token-value>
```

---

## 5️⃣ When Should CSRF Protection Be Disabled?

CSRF can be safely disabled when:

| Scenario | Reason |
|----------|--------|
| **Stateless REST APIs (JWT)** | No session cookies = no CSRF risk |
| **Mobile / 3rd-party clients** | They don't run in browsers |
| **Public APIs** | No user-authenticated state |
| **Token-based auth via headers** | Tokens not auto-sent like cookies |

```java
http.csrf(csrf -> csrf.disable());
```

⚠ **Never disable CSRF** for traditional cookie-based browser apps.

---

## 6️⃣ What is `@CrossOrigin` Annotation?

A shortcut to **enable CORS at controller or method level**.

```java
@CrossOrigin(origins = "https://frontend.com",
             methods = {RequestMethod.GET, RequestMethod.POST},
             allowedHeaders = "*",
             maxAge = 3600)
@RestController
public class ProductController { }
```

- Applied at class → affects all endpoints.
- Applied at method → affects only that endpoint.

---

## 7️⃣ Authentication vs Authorization

| Aspect | Authentication | Authorization |
|--------|----------------|----------------|
| Question Answered | "Who are you?" | "What can you do?" |
| Order | First | After authentication |
| Mechanism | Username/password, JWT, OAuth | Roles, permissions |
| Spring Component | `AuthenticationManager` | `AccessDecisionManager` / `AuthorizationManager` |
| Example | Login | Access to `/admin` page |

---

## 8️⃣ What is JWT Authentication?

**JWT (JSON Web Token)** = a self-contained token used for **stateless authentication**.

### 🔹 Structure
```
HEADER.PAYLOAD.SIGNATURE
```
Example:
```
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ1c2VyMSJ9.SflKxw...
```

| Part | Contents |
|------|----------|
| Header | algorithm, token type |
| Payload | claims (user id, roles, expiry) |
| Signature | HMAC of header+payload using secret |

### 🔹 Flow
1. User logs in → server returns JWT.
2. Client stores JWT (localStorage/header).
3. Each request includes:
   ```
   Authorization: Bearer <token>
   ```
4. Server validates signature & expiry → grants access.

✅ No server-side session needed → highly scalable.

---

## 9️⃣ JWT vs Session-Based Authentication

| Feature | Session-Based | JWT |
|---------|---------------|-----|
| Storage | Server (in-memory/Redis) | Client (token) |
| State | Stateful | Stateless |
| Scalability | Harder (sticky sessions) | Easy |
| Revocation | Easy (just remove session) | Hard (token valid till expiry) |
| Transport | Cookies | HTTP Header (usually) |
| CSRF Risk | High | Low (when in header) |
| Use Case | Monoliths | Microservices, SPAs, Mobile |

---

## 🔟 What is Remember-Me Authentication?

Allows users to **stay logged in** across sessions.

### 🔹 Mechanism
1. On login with "Remember Me" checked, server issues a **persistent cookie** (or token).
2. On browser restart, Spring Security recognizes the cookie and auto-authenticates.

### 🔹 Configuration
```java
http.rememberMe(rm -> rm
    .key("uniqueSecret")
    .tokenValiditySeconds(86400));
```

Two strategies:
- **Hash-based token** (default)
- **Persistent token (DB-backed)** — more secure

---

## 1️⃣1️⃣ What is Session Fixation?

A vulnerability where an attacker **sets a known session ID** in the victim's browser before login, and after login, uses that same session ID to impersonate the user.

### 🔹 Prevention in Spring Security
Spring Security **automatically rotates** the session ID after login.

```java
http.sessionManagement(sm -> sm
    .sessionFixation().migrateSession());
```

Options:
- `migrateSession` (default)
- `newSession`
- `changeSessionId`
- `none`

---

## 1️⃣2️⃣ How Does Spring Security Manage Sessions?

```java
http.sessionManagement(sm -> sm
    .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)
    .maximumSessions(1)
    .maxSessionsPreventsLogin(true));
```

### 🔹 Session Creation Policies
| Policy | Description |
|--------|-------------|
| `ALWAYS` | Always create |
| `IF_REQUIRED` | Default |
| `NEVER` | Don't create, use if exists |
| `STATELESS` | Never create / use (JWT) |

Spring uses **`HttpSessionSecurityContextRepository`** to persist authentication across requests.

---

## 1️⃣3️⃣ Logout Implementation in Spring Security

```java
http.logout(logout -> logout
    .logoutUrl("/logout")
    .logoutSuccessUrl("/login?logout")
    .invalidateHttpSession(true)
    .deleteCookies("JSESSIONID"));
```

### 🔹 Internal Flow
1. Logout request hits `LogoutFilter`.
2. Invokes registered **`LogoutHandler`s** (clear cookies, invalidate session).
3. Calls `LogoutSuccessHandler` to redirect or return response.

For JWT → server can blacklist tokens; client deletes stored token.

---

## 1️⃣4️⃣ What is `SecurityContextHolder`?

The **container** that holds the currently authenticated user info **per thread**.

```java
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
String username = auth.getName();
```

### 🔹 Storage Strategies
| Strategy | Description |
|----------|-------------|
| `MODE_THREADLOCAL` (default) | One per thread |
| `MODE_INHERITABLETHREADLOCAL` | Inherited by child threads |
| `MODE_GLOBAL` | Single global context |

---

## 1️⃣5️⃣ Testing Secured APIs in Spring Boot

```java
@SpringBootTest
@AutoConfigureMockMvc
class SecureControllerTest {

    @Autowired MockMvc mvc;

    @Test
    @WithMockUser(username = "admin", roles = "ADMIN")
    void adminAccess() throws Exception {
        mvc.perform(get("/admin"))
           .andExpect(status().isOk());
    }

    @Test
    void unauthenticatedAccess() throws Exception {
        mvc.perform(get("/admin"))
           .andExpect(status().isUnauthorized());
    }
}
```

Useful annotations:
- `@WithMockUser`
- `@WithAnonymousUser`
- `@WithUserDetails`

---

## 1️⃣6️⃣ What is `@WithMockUser`?

Simulates an **authenticated user** in tests **without going through login**.

```java
@Test
@WithMockUser(username = "john", roles = {"USER"})
void getUserProfile() throws Exception { ... }
```

➡ Internally populates `SecurityContextHolder` with a mock `Authentication` object before the test runs.

---

## 1️⃣7️⃣ How Do Filters Work Internally in Spring Security?

Spring Security is built on the **Servlet Filter Chain** pattern.

### 🔹 Architecture
```
Client → DelegatingFilterProxy → FilterChainProxy → [Security Filters] → DispatcherServlet
```

### 🔹 Common Filters (Order Matters)
1. `SecurityContextPersistenceFilter`
2. `UsernamePasswordAuthenticationFilter`
3. `BasicAuthenticationFilter`
4. `BearerTokenAuthenticationFilter` (JWT)
5. `CsrfFilter`
6. `CorsFilter`
7. `ExceptionTranslationFilter`
8. `FilterSecurityInterceptor` / `AuthorizationFilter` (final check)

Each filter:
- Inspects the request
- Modifies `SecurityContext`
- Either lets it pass or rejects it

### 🔹 Add Custom Filter
```java
http.addFilterBefore(new JwtAuthFilter(),
       UsernamePasswordAuthenticationFilter.class);
```

---

## 1️⃣8️⃣ `OncePerRequestFilter` vs `GenericFilterBean`

| Feature | `OncePerRequestFilter` | `GenericFilterBean` |
|---------|------------------------|---------------------|
| Executes | **Once per HTTP request** | May execute multiple times (e.g., on forwards) |
| Best for | Authentication, logging | Generic filters |
| Method | `doFilterInternal()` | `doFilter()` |
| Recommended | ✅ For Spring apps | ❌ Less safe |

### 🔹 Example
```java
public class JwtAuthFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest req,
                                    HttpServletResponse res,
                                    FilterChain chain) {
        // validate JWT
        chain.doFilter(req, res);
    }
}
```

---

# ☁️ PART 2 — Spring Cloud & Microservices

---

## 1️⃣9️⃣ What is Spring Cloud?

**Spring Cloud** is a **suite of tools** built on top of Spring Boot that helps developers build **distributed, cloud-ready microservices**.

### 🔹 Provides Solutions For
| Concern | Tool |
|---------|------|
| Service discovery | Eureka |
| Configuration management | Spring Cloud Config |
| API Gateway | Spring Cloud Gateway |
| Load balancing | Spring Cloud LoadBalancer |
| Circuit breaking | Resilience4j |
| Distributed tracing | Sleuth / Micrometer Tracing |
| Messaging | Spring Cloud Stream |

---

## 2️⃣0️⃣ Spring Boot vs Spring Cloud

| Feature | Spring Boot | Spring Cloud |
|---------|-------------|--------------|
| Purpose | Build single standalone app | Build distributed microservices |
| Scope | One service | Multiple coordinated services |
| Includes | Auto-config, embedded server | Discovery, config, gateway, etc. |
| Depends on | Standalone | **Built on top of Spring Boot** |

➡ Spring Cloud **uses** Spring Boot under the hood.

---

## 2️⃣1️⃣ Why is Service Discovery Important?

In a microservices system, instances change dynamically (auto-scaling, deployments).

### 🔹 Without Service Discovery
- Hardcoded URLs → unsustainable.
- Difficult to scale.
- Manual updates required.

### 🔹 With Service Discovery
- Services **register themselves** with a registry.
- Clients **look up services by name**, not URL.
- Enables **load balancing**, **fault tolerance**, **dynamic scaling**.

---

## 2️⃣2️⃣ What is Eureka Server & Eureka Client?

**Eureka** = Netflix's open-source **service registry**.

### 🔹 Eureka Server
- A central registry of all microservices.
- Holds metadata (host, port, health, etc.)

```java
@EnableEurekaServer
@SpringBootApplication
public class EurekaServerApp { }
```

### 🔹 Eureka Client
- A microservice that registers itself with the Eureka Server.

```java
@EnableDiscoveryClient
@SpringBootApplication
public class OrderServiceApp { }
```

```yaml
spring.application.name: order-service
eureka.client.serviceUrl.defaultZone: http://localhost:8761/eureka/
```

---

## 2️⃣3️⃣ How Does the Eureka Heartbeat Mechanism Work?

1. On startup, **client registers** itself with the Eureka server.
2. Client sends **heartbeats every 30s** (default) to indicate it's alive.
3. Eureka server expects heartbeats; if missing for **90s**, marks instance as DOWN.
4. After grace period, removes instance from registry.
5. Eureka also has **self-preservation mode** — if too many heartbeats fail (network issues), it stops evicting to avoid mass deregistration.

```yaml
eureka.instance.lease-renewal-interval-in-seconds: 30
eureka.instance.lease-expiration-duration-in-seconds: 90
```

---

## 2️⃣4️⃣ What is an API Gateway?

An **API Gateway** = single entry point for all client requests in a microservices system.

### 🔹 Responsibilities
- **Routing** requests to the right service
- **Authentication / Authorization**
- **Rate limiting**
- **Load balancing**
- **Logging, tracing, monitoring**
- **Request/response transformation**

### 🔹 Example (Spring Cloud Gateway)
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/orders/**
```

Popular gateways: **Spring Cloud Gateway**, **Zuul** (older), **Kong**, **Nginx**.

---

## 2️⃣5️⃣ What is a Config Server?

**Spring Cloud Config Server** = centralized configuration server.

### 🔹 Purpose
- Store and serve external configuration for all microservices.
- Configs typically stored in **Git**, file system, or Vault.
- Supports **profile-specific** and **versioned** configs.

### 🔹 Setup
```java
@EnableConfigServer
@SpringBootApplication
public class ConfigServerApp { }
```

```yaml
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/example/configs
```

### 🔹 Client
```yaml
spring.config.import: optional:configserver:http://localhost:8888
```

➡ Enables **runtime refresh** with `@RefreshScope` + Actuator's `/refresh`.

---

## 2️⃣6️⃣ Feign Client vs RestTemplate

| Feature | RestTemplate | Feign Client |
|---------|--------------|--------------|
| Style | Imperative | Declarative |
| Boilerplate | High | Low (interfaces) |
| Integration | Basic | Eureka + Ribbon/LoadBalancer + Resilience4j |
| Status | Deprecated (Spring 5+) | Recommended for microservices |
| Async | Limited | Yes |

### 🔹 RestTemplate Example
```java
restTemplate.getForObject("http://order-service/orders/1", Order.class);
```

### 🔹 Feign Example
```java
@FeignClient(name = "order-service")
public interface OrderClient {
    @GetMapping("/orders/{id}")
    Order getOrder(@PathVariable Long id);
}
```

➡ Modern alternative: **`WebClient`** (reactive, non-blocking).

---

## 2️⃣7️⃣ What is the Circuit Breaker Pattern?

Prevents a **failing service** from cascading failures across the system.

### 🔹 States
```
CLOSED  → all requests go through
OPEN    → fail-fast, calls blocked
HALF-OPEN → limited trial requests
```

### 🔹 Flow
1. Service A calls Service B.
2. If B fails repeatedly → circuit breaker **opens**.
3. A immediately returns fallback without calling B.
4. After cooldown → moves to **half-open** to test.
5. If success → **closes**; else stays open.

➡ Avoids **resource exhaustion** and improves **resiliency**.

---

## 2️⃣8️⃣ Problems Resilience4j Solves

**Resilience4j** = lightweight fault-tolerance library (replaces Hystrix).

| Problem | Module |
|---------|--------|
| Cascading failures | **CircuitBreaker** |
| Slow responses | **TimeLimiter** |
| Transient errors | **Retry** |
| Overload | **RateLimiter** |
| Resource exhaustion | **Bulkhead** |
| Caching results | **Cache** |

### 🔹 Example
```java
@CircuitBreaker(name = "paymentService", fallbackMethod = "fallback")
@Retry(name = "paymentService")
public String callPayment() { ... }

public String fallback(Throwable t) {
    return "Payment service temporarily unavailable";
}
```

✅ Lightweight, modular, integrates with **Micrometer** for monitoring.

---

## 2️⃣9️⃣ What is Centralized Configuration in Microservices?

A pattern where **all configuration is stored in one place** instead of inside each service.

### 🔹 Benefits
- 🔁 Single source of truth
- 🌍 Environment-specific configs (dev, qa, prod)
- 🔒 Secure secret management
- 🚀 Runtime refresh without redeploy
- 📦 Version control via Git

### 🔹 Common Tools
- Spring Cloud Config
- HashiCorp Consul
- AWS Parameter Store / Secrets Manager
- Kubernetes ConfigMaps & Secrets

---

## 3️⃣0️⃣ How Do Microservices Communicate Securely?

Microservices typically communicate over the network, so security is essential.

### 🔹 Common Security Mechanisms

| Mechanism | Description |
|-----------|-------------|
| **HTTPS / TLS** | Encrypt traffic in transit |
| **mTLS (Mutual TLS)** | Both sides authenticate via certificates |
| **JWT / OAuth2** | Token-based service auth |
| **API Gateway Auth** | Single point validates tokens |
| **Service Mesh** (Istio, Linkerd) | Auto mTLS, traffic policies |
| **Network Segmentation** | Internal-only services in private subnets |
| **Secrets Management** | Vault, AWS Secrets Manager |
| **Rate Limiting & Throttling** | Prevent abuse |
| **Message-Level Security** | Sign / encrypt messages (Kafka, RabbitMQ) |

### 🔹 Typical Pattern
```
Client → API Gateway (JWT validation)
         │
         ▼
   Microservices (mTLS between them)
         │
         ▼
   Database / Storage (encrypted)
```

---

# 🧠 Quick Recap Cheatsheet

### Security
| Concept | Key Idea |
|---------|----------|
| CORS | Browser-enforced cross-origin policy |
| CSRF | Forging requests using user session |
| JWT | Stateless self-contained token |
| SecurityContextHolder | Stores current user's auth info |
| Filters | Backbone of Spring Security |
| `@WithMockUser` | Simulate user in tests |

### Microservices
| Concept | Tool |
|---------|------|
| Service Discovery | Eureka |
| API Gateway | Spring Cloud Gateway |
| Config | Spring Cloud Config |
| Circuit Breaker | Resilience4j |
| Inter-service Call | Feign / WebClient |
| Centralized Config | Git-backed Config Server |
| Secure Comm | HTTPS, mTLS, JWT, OAuth2 |

