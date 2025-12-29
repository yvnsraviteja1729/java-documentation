<p><a target="_blank" href="https://app.eraser.io/workspace/vdXgGjnIc7O0pvycwfEg" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>



# Aspect Oriented Programming (AOP)
- Helps to intercept method invocation and we can perform some tasks before and after method invocation.
- AOP allows you to handle boilerplate code and repeated code like logging, transaction management and allows you to focus on business logic
- Aspect module handles boilerplate
- Helps in achieving re usability
- Used during Logging , Transaction Management , Security etc






# Pointcut Expression:
Expression which tells where an advise should be applied.



# Types Of Pointcut Expression:
# 
1. **Execution** - Matches Method In Class
    1. * wildcard - matches 1 item
    2. (..) wildcard - matches 0 or more item


2. **Within - Matches all method within any class or package
**
3. @within - matches any method in a class which has this annotation 

4. @annotation - matches any method that is annotated with given annotation 

5. Args - matches any method with particular arguments

6.  @args - matches any method with particular parameter and that parameter class is annotated with particular annotation

7. target - matches any method on particular instance of a class


# Combination Of Pointcuts Using:
- && boolean and
- || boolean or


# Named Pointcuts
Save pointcuts to method and use it



# **Advice:
- It is an action
- Which is taken @Before or @After or @Around the method execution  


Questions :

- How aspect work under the hood?
- How Spring manages invocation of aspect functions?
- What will happen if we have 1000 aspects?
- what will happen to latency if we have more aspects?












# Spring Transactions & @Transactional Annotation 


## Critical Operations or Code Sections
Code segment where shared resources are being accessed and modified

```
_{_
_Read Flight Seat With ID 1_
_     if SeatStatus is Available_
_           update to booked_
_}_
```
When multiple requests try to access this critical section, data inconsistency can happen

What is the solution??





# Transaction 


Helps to achieve **ACID** properties

## A (Atomicity)
Ensures all operations in transaction are completed successfully. If any operation fails then entire transaction is rolled back.

## C(Consistency)
Ensures the DB state before and after transaction in consistent 

## I (Isolation)
Ensures that , if multiple transactions are running in parallel, they do not interfere with each other

## D(Durability)
Ensures that committed transaction will never be lost despite of system failures









```
BEGIN_TRANSACTION
          - Debit From A
          - Credit To B
          If All Success
                COMMIT;
            Else:
                ROLLBACK;
END_TRANSACTION
```






## Transactional Context
## Transaction Managers
1. Programmatic
2. Declarative


## Transaction Propagation
- `REQUIRED` 
- `SUPPORTS` 
- `MANDATORY` 
- `REQUIRES_NEW` 
- `NOT_SUPPORTED` 
- `NEVER` 
- `NESTED` 


## Isolation Levels
- `DEFAULT` 
- `READ_UNCOMMITTED` 
- `READ_COMMITTED` 
- `REPEATABLE_READ` 
- `SERIALIZABLE` 


## Transaction Timeouts
## Read Only Transactions 




# Transaction Managers
- Core component in Transactions
- Responsible for managing transactions
## 1 Declarative
- Transaction Management Through Annotations
- @Transactional 
- Spring boot will choose appropriate transaction manager
-  We can create explicit transaction manager and tell spring to choose that
## 2 Programmatic
- Handle Transaction programmatically through code
- Flexible but difficult to maintain
- Which case to use this? Very specific case
```
﻿@Component
public class User {
  @Transactional
    public void updateUser(){
      //1. Update DB
      //2. External API Call
      //3. Update DB
      }
  } 
```


2 Approaches for Programmatic 



## Transaction Propagation
- `REQUIRED - Done` 
- `SUPPORTS``- Done` 
- `MANDATORY``- Done` 
- `REQUIRES_NEW``- Done` 
- `NOT_SUPPORTED``- Done` 
- `NEVER``- Done` 
- `NESTED``- Done` 


Transaction propagation refers to how transactions behave when a method annotated with `@Transactional` is called by another method that may or may not have an active transaction.



```csharp
1✅ Java OOPs concepts
2✅ Java Memory Model
3✅ Java Garbage Collection
4✅ Keywords:
 static,
 final,
 volatile,
 synchronized,
 interface
5✅ Java 8 Features - Stream, Lambda,
6✅ Functional Interface
7✅ Comparator
8✅ Multi Threading
9✅ Junit & Mockito
10✅ Spring Boot CRUD
11✅ Important Java Annotation and their use
12✅ Important Spring Annotation and their use
13✅ Spring JPA
14✅ Entity relationships - 1to1, 1toMany etc
15✅ Basic SQL Queries
16✅ Design Patterns - singleton, factory,
decorator, observer
17✅ Use of caching
18✅ Microservices Concepts - 
RateLimiter, Api Gateway,
 load balancer, scaling


```
```xml
﻿1:✅Tell me about yourself.
2:✅Why do you want to work for our company?
3:✅ What are your greatest strengths and weaknesses?
4:✅Where do you see yourself in five years?
5:✅Why are you leaving your current job?
6:✅Describe a challenging work situation and how you overcame it
7:✅How do you handle stress and pressure?
8:✅What are your salary expectations?
9:✅Do you have any questions for us?
10:✅What motivates you in your work? 
```
# Spring Beans & Application Context
- [x] What is Bean?
- [x] What is Application Context?
- [x] How to Create Beans?32
- [x] How to check if the beans are present in application?
    - [x] 1. Using Actuators
    - [x] 2.Using object of application context



Bean Is a Java Object

What Is Spring IoC Container?



2 Ways - Creating Bean

1. @Component 
2.  @Configuration & @Bean 
# @ComponentScan Annotation 
- [x] What is component scanning?
- [x] Use of @Component Scan Annotation
- [x] ComponentScan arguments
    - [x] basePackage
    - [x] exclusions /excludeFilters





# @Primary & @Qualifier Annotations 
- [ ] What is problem ?
- [ ] Solution By using annotations
- [ ] Use of @Primary and @Qualifier 
- [ ] @Primary vs @Qualifier 
- [ ] When to use @Primary and when to use @Qualifier 
 





**When to use **`**@Primary**` **:**

- Use `@Primary`  when you want to designate one of the beans as the default choice for dependency injection when there are multiple beans of the same type. In this case, `DefaultPaymentService`  is marked as the primary bean, so whenever Spring needs a `PaymentService`  and no `@Qualifier`  is specified, it will inject `DefaultPaymentService` .
- **Example Usage**: In the CheckoutService , since no `@Qualifier`  is used, Spring will inject the `DefaultPaymentService`  due to the `@Primary`  annotation.
**When to use **`**@Qualifier**` **:**

- Use `@Qualifier`  when you need to specify a particular bean to be injected when there are multiple beans of the same type. It provides more explicit control over which bean should be used in a particular context.
- **Example Usage**: In the `CheckoutService` , the `@Qualifier("GPayService")`  annotation ensures that the `GPayService`  bean is injected, even though `DefaultPaymentService`  is marked as primary.










#  Exception Handler & Controller Advise
- [ ] Traditional way of handling exceptions
- [ ] Exception Handler 
- [ ] Need Of Controller Advise and Implementation








# Customizing the Nature of a Bean In Lifecycle


- [ ] What Are Custom Actions While Bean Creation
- [ ] Interfaces to customize bean Nature
    - [ ] `InitializingBean` 
    - [ ] `DisposableBean` 

- [ ] `@PostConstruct`  and `@PreDestroy`  annotations
- [ ] [﻿View on canvas](https://app.eraser.io/workspace/vdXgGjnIc7O0pvycwfEg?elements=y_jaFOapZXZ4uPCdePIwZw) 




















## What Are Custom Actions While Bean Creation
- let the bean perform certain actions upon initialization and destruction of your beans.
- Perform Actions After Bean Creation
- Perform Actions Before Destroying Bean


# @Aspect Annotation 
- [x] Aspect Oriented Programming (Spring AOP)
- [x] Terminologies in AOP
- [x] Aop Proxies
- [x] Pointcut Expressions
    - [x] Before
    - [x] After
    - [x] Around

- [ ] Pointcut Designators






# Transaction Isolation Levels
- `DEFAULT` 
- `READ_UNCOMMITTED` 
- `READ_COMMITTED` 
- `REPEATABLE_READ` 
- `SERIALIZABLE` 


# Logging In Spring Boot
- Logging
- Logback & Log4j Intro
- Logging levels
- Enable logging using SLF4j annotation


| Level |   |
| ----- | ----- |
| `FATAL`  | HIGH |
| `ERROR`  | HIGH |
| `WARN`  | MEDIUM |
| `INFO`  | NORMAL |
| `DEBUG`  | NORMAL |
| `TRACE`  | NORMAL |












# Unit Testing In Spring Boot Applications
- [ ] Introduction to unit testing
- [ ] Why Unit Tests Are Important
- [ ] Getting Started With UT In Spring Boot App
- [ ] Write Test Cases Using Junit5 & Mockito


























### **What Is Unit Test**:
A **unit test** is a way to test the smallest parts of your code, such as individual methods or classes, to ensure they work as expected



### ** Why Unit Testing is Important**:
- **Fast feedback**: Unit tests are quick to run and can immediately tell you if something is broken.
- **Early bug detection**: Catch bugs at the method level before they propagate.
- **Easier to maintain**: They help ensure future changes don’t break existing functionality.
- **Increases confidence**: You can refactor your code with the confidence that everything still works.






# Unit Testing In Spring Boot Applications Part 2


- [ ] Other Useful Annotation in Test lifecycle
- [ ] `@BeforeAll` , `@BeforeEach` , `@AfterAll` , `@AfterEach` 
- [ ] Mocking void methods using `doNothing()` & verify
- [ ] Testing private methods 
- [ ] Testing exceptional scenarios




# 
| **Feature** | **Abstract Class** | **Interface** |
| ----- | ----- | ----- |
| **Fields (State)** | **Can have fields (variables) to store state.** | **Cannot have instance fields, only constants.** |
| **Methods** | **Can have both abstract and non-abstract methods.** | **Can have abstract and default methods.** |
| **Constructors** | **Can have constructors to initialize fields.** | **Cannot have constructors.** |
| **Multiple Inheritance** | **Can only extend one abstract class.** | **Can implement multiple interfaces.** |
| **Use Case** | **Use when you want to share state between subclasses ** | **Use to define common behavior across unrelated classes ** |






























API





















# API


## **Application Programming Interface**
it’s a set of rules or protocols that allows two applications to communicate with each other.



APIs make it easier to connect different software systems, saving time and allowing each system to specialize in what it does best





















# String Immutability & String Pools In Java
- [x] What is String Immutability?
- [x] What is the String Pool?
- [x] Why Strings are immutable?
- [x] String Interning 
- [x] Common Pitfalls and Best Practices 
- [x] Interview Questions 






**Why Strings are Immutable**:

- "The immutability of Strings has some key advantages:
    - **Thread Safety**: Since Strings cannot be modified, they are inherently thread-safe. Multiple threads can use the same String without worrying about data corruption.
    - **HashCode Consistency**: Strings are widely used as keys in `HashMap`  or `HashSet` . Immutability ensures the hashcode of a String doesn’t change after it’s created.
    - **Security**: Strings are used in sensitive operations like class loading or database connection URLs. Immutability ensures they cannot be modified maliciously."





## Common Pitfalls and Best Practices
**Using **`**new String()**`** unnecessarily**

**String Concatenation**

**Null Strings**













# Spring Batch
- [ ] What & Why Spring Batch?
- [ ] Spring Batch Architecture
**Using Spring Batch:** With Spring Batch, you can handle this efficiently and reliably by breaking it down into manageable steps:

1. **Chunk Processing**: Spring Batch reads transactions for a few hundred customers at a time, processes them in chunks, and writes them to the statements in bulk.
2. **Scheduled Execution**: Set up the job to run during off-peak hours (like late at night), reducing the impact on the bank’s primary systems.
3. **Error Handling and Retry**: If there’s an error while processing a batch, Spring Batch can automatically retry the step or skip the problematic data and continue with the rest, logging any issues for further review.
4. **Restartability**: If a job fails midway through generating 10,000 statements, Spring Batch can restart from where it left off, ensuring every customer receives their statement without duplication.
5. **Job Monitoring**: With its built-in Job Repository, Spring Batch tracks each job’s progress and stores metadata on completed, in-progress, and failed jobs, allowing easy monitoring and troubleshooting.










# Spring Batch PART 2
- [ ] Implement Spring Batch in Spring Boot App
- [ ] Create a job to insert large data








# Spring Data JPA 


## 1 Introduction 
- What is JDBC?
- What is ORM?
- What is JPA ?


## 2. JPA Architecture & Entity Lifecycle
- Architecture of JPA
- Entity Lifecycle 
- Understand Internals of JPA


## 3. Spring Data JPA Implementation 
- Create Entities Example
- Spring Data JPA Annotations
- Repositories 
    - `CrudRepository` +  `ListCrudRepository` 
    - `**JpaRepository**` 
    - `**PagingAndSortingRepository**` +`**ListPagingAndSortingRepository**` 

- Inbuilt methods for CRUD
-  Query Methods in Advance
- @Query Annotations in Advance with 4 or 5 important examples.










| Feature | CrudRepository | PagingAndSortingRepository | JpaRepository |
| ----- | ----- | ----- | ----- |
| Basic CRUD Operations | ✅ | ✅ | ✅ |
| Pagination | ❌ | ✅ | ✅ |
| Sorting | ❌ | ✅ | ✅ |
| JPA-Specific Methods | ❌ | ❌ | ✅ |
| Custom Query Methods | ❌ | ❌ | ✅ |
| Batch Operations | ❌ | ❌ | ✅ |
|  |  |  |  |




| Feature | CrudRepository | PagingAndSortingRepository | JpaRepository |
| ----- | ----- | ----- | ----- |
| Basic CRUD Operations | ✅ | ❌ | ✅[﻿View on canvas](https://app.eraser.io/workspace/vdXgGjnIc7O0pvycwfEg?elements=0eqEB_nc6DbWMEZq6lASSg)  |
| Pagination | ❌ | ✅ | ✅ |
| Sorting | ❌ | ✅ | ✅ |
| JPA-Specific Methods | ❌ | ❌ | ✅ |
| Custom Query Methods | ❌ | ❌ | ✅ |
| Batch Operations | ❌ | ❌ | ✅ |
|  |  |  |  |












# Java Basics - Classes & Objects




Object Oriented Programming :

Objects - real world entities - car , bike, house, person

properties - brand, speed.  -Attributes/fields

behavior - start() - Function





# Java Basics - OOPS Concepts
**Inheritance** - "Reusing and extending existing code by inheriting properties and behavior."

**Polymorphism** - "Allowing one interface to represent different underlying forms or types."

**Encapsulation** - "Hiding the internal details of objects and exposing only what’s necessary."

**Abstraction** - "Focusing only on essential details while hiding the complex implementation."







## Purpose and Benefits of Inheritance 
1.   Code Reusability
2.    Extensibility 
3.   Maintainability


# **Polymorphism**
In programming, polymorphism allows the same action or method to work in different ways depending on the object that uses it.



**Overloading = Compile-Time Polymorphism** (decision made at compile time).

**Overriding = Runtime Polymorphism** (decision made at runtime).



**Different Parameter List:**
The **number, type, or order of parameters** must differ.

- **Number:** `add(int a)`  vs. `add(int a, int b)` 
- **Type:** `add(int a)`  vs. `add(double a)` 
- **Order:** `add(int a, double b)`  vs. `add(double a, int b)` 
**Return Type Doesn't Matter:**

void show() {} // Valid 

int show() {} // Compiler error: Duplicate method



### **Code Reusability**
Polymorphism allows you to reuse existing code by writing methods that work with a parent class or interface, without knowing the specific subclass.



### **Flexibility and Extensibility**
You can introduce new classes or functionality into a program without altering the existing code.



### **Easier Maintenance**
By using polymorphism, you centralize shared behavior in a base class while allowing specific subclasses to provide custom implementations. 

### **Improved Readability and Modularity**
Polymorphism encourages clean code by organizing related behaviors into a hierarchy. This modular approach makes the code easier to read and understand.









# Spring Interceptors
- [ ] What Are Interceptors?
- [ ] Handler Interceptor
- [ ] Implementation
- [ ] Interceptor vs filters






# Spring Data JPA Relationships 
- [ ] Relationship Between Entities
- [ ] Types Of Relationship
- [ ] OneToOne
- [ ] OneToMany
- [ ] ManyToOne
- [ ] ManyToMany
- [ ] Key considerations for JPA relationships: **Unidirectional vs. Bidirectional, Owning Side and Inverse Side, Fetch Types, Cascade Operations, **
- [ ]  Relationship Annotations




# Encapsulation:
- [x] Packages
- [x] Access Modifiers
- [x] Encapsulation 


## **Packages in Java**
### **What Are Packages?**
- Packages are like folders in a file system that help organize your code.
- They group related classes and interfaces to improve code modularity and reusability.
### **Types of Packages**
- **Built-in Packages**: Java's predefined packages (e.g., `java.util` , `java.io` ).
- **User-Defined Packages**: Custom packages created by developers.


**Access Modifiers**

- Public
- Protected
- Default
- Private


| **Modifier** | **Within Class** | **Within Package** | **Outside Package (Subclass)** | **Outside Package** |
| ----- | ----- | ----- | ----- | ----- |
| Private | ✔ | ❌ | ❌ | ❌ |
| Default | ✔ | ✔ | ❌ | ❌ |
| Protected | ✔ | ✔ | ✔ | ❌ |
| Public | ✔ | ✔ | ✔ | ✔ |








# Spring Boot Caching
- [ ] What is cache?
- [ ] Cache In Spring Boot
- [ ] Types of Caching  
    - [ ] in Memory
    - [ ] Distributed

- [ ] Annotations Used In Caching
    - [ ] @EnableCaching
    - [ ] @Cacheable
    - [ ] @CachePut
    - [ ] @CacheEvict
    - [ ] @Caching





**Why Use Caching?**

- Reduce latency: Faster access to data compared to fetching from a database or an external service.
- Decrease load: Reduces the number of calls to the backend systems or database.
- Improve scalability: Helps applications handle higher traffic loads efficiently.




Spring Boot Caching - Redis

- [ ] What is Redis?
- [ ] Connect Spring Boot App To Redis
























# Spring Boot Scheduling 




**What is scheduling?**

- Scheduling in Spring Boot allows you to **automate tasks** at specific intervals or times
- It helps in running background jobs without manual intervention.
**Why Do We Need Scheduling?**

- Without scheduling, repetitive tasks must be **triggered manually**
### **💡 Use Cases of Scheduling**
| **Scenario** | **How Scheduling Helps** |
| ----- | ----- |
| **Order Processing** | Automatically process pending orders every 5 minutes |
| **Email Notifications** | Send promotional emails daily at midnight |
| **Data Cleanup** | Delete old records every Sunday at 2 AM |
| **Stock Updates** | Fetch stock prices from APIs every hour |
| **System Health Check** | Log system status every minute |




# Spring Boot Dynamic Scheduling 
- [x] What & Why Dynamic Scheduling?
- [x] `ThreadPoolTaskScheduler` 
- [x] Implementation








# Spring Events (Advanced Features)
- [x] Order In Listeners
- [x] Async Event Listeners
- [x] Conditional Event Listeners
- [x] Global Error Handling In Listeners






# AWS CLI
- [x] What Is AWS CLI?
- [x] How To Install AWS CLI?
- [x] Configure AWS CLI
- [x] How To Use AWS CLI?




# AWS API Gateway
- [x] API Gateway ?
- [x] Hands On Demo


👉 Amazon API Gateway is a fully managed service that lets you create, publish, secure, and monitor APIs at scale. It acts as a ‘front door’ for applications to access data, business logic, or services running on AWS.



- **Routing** – forwards request to the right backend (Lambda, EC2, DynamoDB, etc.).
- **Security** – integrates with IAM, Cognito, API keys.
- **Rate Limiting & Throttling** – protect backend from overload.
- **Monitoring** – integrates with CloudWatch.
- **Transformations** – request/response mapping (JSON/XML changes).
- **Caching** – improve performance.




# Trigger Lambda From API Gateway
- [x] Create a lambda function
- [x] Create API In Api Gateway
- [x] Trigger Lambda Function From API Gateway




# AWS SSM Parameter Store


- [ ] 














# AWS SQS (Simple Queue Service)
- [x] What Is SQS ?
- [x] Queue Types
- [x] Features
- [x] Hands On Demo
- [x] Use Cases




##  **What is SQS?**
- Amazon **Simple Queue Service** is a **fully managed message queueing service**.
- It helps **decouple microservices, distributed systems, and serverless applications**.
- Core idea: _Producer → Queue → Consumer_
- No direct connection between producer and consumer — they communicate via queue.






## Use Cases
### 1. **Order Processing in E-commerce**
- Customers place orders → requests go into an **OrderQueue**.
- Workers pick up orders one by one and process them (inventory check, payment, shipping).
 ✅ Ensures **no order is lost** even if the system is busy.
 ✅ Handles **high spikes on sale days** (like Flipkart Big Billion Day / Amazon sale).
---

### 2. **Decoupling Microservices**
- In a microservices setup:
    - **User Service** creates a user and puts a message in SQS.
    - **Email Service** reads the message and sends a welcome email.

- If the email service is down, messages wait in SQS until it’s back.
 ✅ Services are **loosely coupled** and don’t depend on each other being live.
---

### 3. **Video / Image Processing**
- A user uploads a video.
- Upload service pushes a message into **VideoProcessingQueue**.
- Background workers read the queue and process the video (compression, thumbnail creation, etc.).
 ✅ Handles **long-running background jobs**.
 ✅ Scales workers **up and down** easily.
---

### 4. **Payment Transactions**
- When you pay via UPI / card:
    - Payment request goes into **PaymentQueue**.
    - Worker processes payment with retry logic.
 ✅ Avoids **duplicate processing** (because of visibility timeout).
 ✅ Ensures **reliable transaction handling** even if bank API is slow.







# AWS SNS (Simple Notification Service)
- [x] What Is SNS ?
- [x] Key Features
- [x] Hands On Demo
- [x] Use Cases




# 📌 What is Amazon SNS?
Amazon **SNS** is a **fully managed messaging service** that helps you send messages or notifications from one place to many systems, apps, or users at the same time.

Think of it as a **megaphone** 📢:

- You (publisher) speak into it once,
- Everyone who has subscribed (subscribers) hears the message instantly.




## **Use Cases of SNS**
1. **Push notifications to users** (SMS/email alerts).
 👉 e.g., OTPs, delivery updates.
2. **Microservices eventing** – decouple services.
 👉 Order Service publishes → Inventory, Billing, and Notification services subscribe.
3. **Fan-out messaging** – One event → multiple systems.
 👉 Upload a file → trigger Lambda, notify via email, push to analytics.






# Github Copilot 
- [x] What is Copilot?
- [x] Integration With Intellij IDE
- [x] Copilot Modes - Chat, Autocomplete & Agent
- [x] Free & Paid Plans








# AWS Secret Manager
- [x] What is Secret Manager?
- [x] Features Of Secret Manager
- [x] Demo


**AWS Secrets Manager** is a fully managed service that helps you securely store, manage, and retrieve **secrets** like database credentials, API keys, OAuth tokens, and other sensitive configuration data.

It automatically handles **encryption, rotation, fine-grained access control (via IAM)**, and integrates easily with AWS services and applications. Instead of hardcoding credentials in your code, you fetch them securely at runtime from Secrets Manager.



###  Features of Secrets Manager
- **Secure Storage**: Stores DB creds, API keys, OAuth tokens, SSH keys.
- **Automatic Rotation**: For supported DBs like RDS MySQL, PostgreSQL, Aurora.
- **Tight IAM Integration**: Control who can access which secret.
- **Audit with CloudTrail**: See who accessed secrets and when.
- **Multi-Region replication**: Secrets can be replicated across regions for global apps.








# Host Static Website On AWS S3
- [x] AWS S3
- [x] Simple Static Website
- [x] Create S3 Bucket
- [x] Upload files
- [x] Configure Bucket For Static Website Hosting
- [x] Set Permissions
- [x] costing




### **1. Storage Costs**
- You pay for **how much data your website stores** in S3.
- S3 Standard: ~$0.023 per GB per month.
- Example: A 50 MB portfolio site costs **almost nothing**, around $0.001 per month.
---

### **2. Data Transfer / Bandwidth**
- You pay for data transferred **from S3 to your users**.
- First 1 GB per month → **Free**.
- 1 GB – 10 TB → ~$0.09 per GB.
- Small sites with ~1,000 visitors/month typically **stay free**.
---

### **3. Requests Costs**
- AWS charges for **requests to your bucket**: GET, PUT, POST, etc.
- GET requests are cheapest (~$0.0004 per 1,000 requests).
- Example: 100,000 GET requests → ~$0.04.
---

### 
### **1. Storage Costs**
- You pay for **how much data your website stores** in S3.
- S3 Standard: ~$0.023 per GB per month.
- Example: A 50 MB portfolio site costs **almost nothing**, around $0.001 per month.
---

### **2. Data Transfer / Bandwidth**
- You pay for data transferred **from S3 to your users**.
- First 1 GB per month → **Free**.
- 1 GB – 10 TB → ~$0.09 per GB.
- Small sites with ~1,000 visitors/month typically **stay free**.
---

### **3. Requests Costs**
- AWS charges for **requests to your bucket**: GET, PUT, POST, etc.
- GET requests are cheapest (~$0.0004 per 1,000 requests).
- Example: 100,000 GET requests → ~$0.04.


---

### **4. Total Estimate for a Small Portfolio**
- Storage: ~$0.001
- Bandwidth: $0 (for ~1 GB/month)
- Requests: ~$0.04
- **Total: ~$0.05/month**
>  “So, hosting a small portfolio on AWS S3 is almost free!” 











# Spring Cloud Config
- [x] Real World Project Problems
- [x] Spring Cloud Config & Architecture
- [x] Config Server Implementation
- [x] Config Client Implementation
- [x] Refresh Properties Using Actuator Refresh




**Spring Cloud Config** is a framework that provides **server-side and client-side support for externalized configuration** in a distributed system.
 It allows you to **store all your application configurations in a central place** (like a Git repository) and lets multiple microservices **fetch their configuration dynamically** from a single source.







# AWS Parameter Store


- [x] Spring Cloud Config Recap
- [x] What Is Parameter Store
- [x] Setup And Demo




**AWS  Parameter Store** is a fully managed service that allows you to **store, manage, and retrieve configuration data and secrets** as key-value pairs.
 It provides a **centralized, secure, and versioned** way to manage environment-specific settings like database URLs, API keys, or feature flags.







# Spring Retry 
- [x] What Is Retry & Usecases?
- [x] Enable Spring Retry
- [x] Retryable Annotation
- [x] Retry & Backoff Config
- [x] Recover Method
- [x] Internal Working Of Spring Retry










# Dependancy Injection Using ObjectProvider, Scoped Proxy, @Lookup
- [x]  Understanding the Core Problem
- [x] Solution #1: ObjectProvider
- [x] Solution #2: Scoped Proxy
- [x] Solution #3: @Lookup Method Injection




**What is ObjectProvider?**

**ObjectProvider is a special Spring injection mechanism that lets you fetch a bean lazily, safely, and dynamically at runtime.**





A **Scoped Proxy** is a Spring-generated proxy object used when a bean’s scope is **shorter-lived** (prototype, request, session) but is injected into a **longer-lived bean** (like a singleton).

A scoped proxy allows Spring to inject a _proxy object_ into the singleton — **not the actual prototype or request bean**.





#  **When to Use Which **
### ✔ **Use ObjectProvider when:**
- You want full control
- You want readable code
- You want to fetch beans manually when needed
- You don’t want proxies involved
### ✔ **Use Scoped Proxy when:**
- You want the easiest solution with **no code change** in singleton
- You want prototype/request/session scoped beans to “just work”
- You prefer DI purity over method overrides
### ✔ **Use @Lookup when:**
- You want a clean **factory-like method**
- You want Spring to generate method code for you
- You want new instance per call without using provider








# @ConfigurationProperties Annotation
 📘 **AGENDA **

- [x] The problem with scattered configuration
- [x] What `@ConfigurationProperties`  actually does
- [x] A simple step-by-step example
- [x] Deep benefits no one talks about
- [x] Nested configs, lists, maps
- [x] Validation
- [x] Best practices










# **1 .The Problem — When @Value Starts Hurting Your Project**
- ❌ Values scattered across the codebase
- ❌ No grouping of related settings
- ❌ No type safety (everything is a String)
- ❌ Refactoring becomes risky
- ❌ Harder to test
- ❌ Harder to maintain consistency across environments


# 2: Enter @ConfigurationProperties 


`@ConfigurationProperties` turns plain configuration into a strongly typed Java object. Instead of injecting one property at a time, you inject a whole configuration bean.



# ** Key Benefits **
1. **Type safety**
 “If the config type mismatches, Spring fails at startup — not at runtime.”
2. **Easier testing**
    - You can simply create an instance of the config class in unit tests.
    - No Spring needed.

3. **Grouping related config**
    - Your properties are logically organized.
    - One bean for one feature.

4. **Supports nested properties**
 – Great for microservices with complex config.
5. **Supports lists and maps**




# @ConfigurationProperties vs @Value — The Real Difference
| Feature | @Value | @ConfigurationProperties |
| ----- | ----- | ----- |
| Type safety | ❌ Weak | ✅ Strong |
| Binding | Binder expression | Automatic object binding |
| Complex objects | ❌ Hard | ✅ Natural |
| Validation | ❌ Manual | ✅ Built-in |
| Testing | ❌ Requires Spring | ✅ POJO testing |
| Use case | One-off property | Grouped config |


# Best Practices
### **1️⃣ Stop Scattering @Value Everywhere**
**"Group related configs. One feature → One config class."**

---

### **2️⃣ Let Java Mirror Your YAML**
**"If it’s nested in YAML, nest it in your config class."**

---

### **3️⃣ Fail Fast With Validation**
**"Add @Validated — catch config errors at startup."**

---

### **4️⃣ Use Lists & Maps—Not Numbered Keys**
**"Spring binds lists & maps automatically. Use them!"**

****

---

### **Prefer Immutable Config**
**"Use constructor binding or records. No accidental changes."**

---









# **LOMBOK **
- [x] What is Lombok
- [x] Lombok in a Spring Boot project
- [x] All major Lombok annotations
- [ ] How Lombok works internally
- [ ] Pros & Cons
- [ ] Why people say Lombok is “heavy”














Lombok is a Java library that reduces boilerplate code by generating common methods like getters, setters, constructors, builders, equals, hashCode, and loggers at compile time.

Lombok lets you focus on business logic instead of repetitive Java code. 





## ✅ PROS: Why Lombok Is Popular
---

### 1️⃣ Massive Boilerplate Reduction
**Biggest win.**

- Removes getters, setters, constructors, builders, loggers
- 50–70% less code in DTOs and services
- Classes become easier to scan
**Example:**

```java
@Data
public class UserDto {
    private Long id;
    private String name;
}
```
👉 Same as ~40 lines of handwritten code.

---

### 2️⃣ Faster Development & Cleaner Code
- Less repetitive typing
- Faster feature delivery
- Developers focus on **business logic**, not plumbing
This is especially valuable in:

- Spring Boot apps
- Microservices
- CRUD-heavy systems
---

### 3️⃣ Zero Runtime Overhead
Important myth-buster:

- No reflection
- No runtime proxies
- No Lombok classes in production
**Generated bytecode = handwritten bytecode**

👉 Performance is **identical**.

---

### 4️⃣ Encourages Best Practices (When Used Correctly)
- Constructor injection via `@RequiredArgsConstructor` 
- Immutable objects via `@Value` 
- Clean logging via `@Slf4j` 
Lombok often nudges teams toward **better design**.

---

### 5️⃣ Mature & Widely Used
- Used by thousands of production systems
- Large community
- Stable for common annotations
Not a toy library.

---

# ❌ CONS: Where Lombok Hurts
---

### 1️⃣ Hidden Code (Reduced Transparency)
This is the **biggest real drawback**.

- Methods are not visible in source code
- Harder to understand behavior at a glance
- Debugging requires mental mapping
Especially problematic for:

- New developers
- Code reviews
- Domain-heavy logic
---

### 2️⃣ IDE Plugin Dependency
Without the Lombok plugin:

- IDE shows red errors
- Code navigation breaks
- Autocomplete fails
👉 Tooling dependency increases cognitive load.

---

### 3️⃣ Uses Compiler Internals (Fragile Design)
Lombok:

- Modifies AST using non-public APIs
- Depends on `javac`  internals
Result:

- Java upgrades can break Lombok
- Preview features may cause issues
- Teams must wait for Lombok updates
This is why some enterprises ban it.



Lombok is **not heavy at runtime**, but it is considered **heavy at compile-time, tooling, and maintenance level**.





# Records Vs Lombok
- [x] What Problem Are We Solving?
- [x] Lombok: What It Is (Quick Recap)
- [x] Java Records: What They Are
- [x] Mutability: The Biggest Difference
- [x] Is Record a replacement of Lombok?
- [ ] When to Use What 

We were writing classes, but thinking in terms of data



immutable data carriers

This class is only about data. Nothing else.





### 👉 What is mutability?
**Mutable object** = its state can change
 **Immutable object** = its state cannot change after creation





Records don’t just reduce boilerplate — they force better design

Why?

- You must think before creating the object
- You model **data**, not behavior
- You separate:
    - Data carriers (Records)
    - Business logic (Services)

This naturally leads to **clean architecture**.





**Immutability Isn’t Always Welcome**

Many frameworks expect:

- No-args constructor
- Setters
- Field mutation


Classes as entities -> Lombok





**When to Use What**

Use **Records** when:

- Data is immutable
- Class is just a data carrier
- You want clarity and safety
Use **Lombok** when:

- Working with JPA entities
- Mutability is required
- You need builders or complex behavior




| Aspect | Lombok | Java Records |
| ----- | ----- | ----- |
| Core Purpose | Reduce boilerplate using annotations | Model immutable data carriers |
| Nature | External library | Language-level feature |
| Introduced In | Pre-Java 8 era (library-based) | Java 16 |
| Design Philosophy | Productivity-first | Design & intent-first |
| Mutability | Mutable by default | Immutable by design |
| Setters | Usually generated | Not allowed |
| Immutability | Optional, developer-controlled | Enforced by compiler |
| Code Generation | Compile-time AST manipulation | Handled by Java compiler |
| Intent Clarity | Implicit (depends on annotations) | Explicit and enforced |
| Debugging Experience | Harder (hidden generated code) | Easier (state never changes) |
| Thread Safety | Developer responsibility | Naturally safer |
| Caching Safety | Risky if mutated | Safe by default |
| Framework Compatibility | Very high | Limited in some cases |
| JPA / ORM Support | Excellent | Poor / not suitable |
| Builders | <p>Supported (</p><p>)</p> | Not supported |
| Custom Behavior | Fully supported | Limited |
| Learning Curve | Easy to start, tricky internally | Easy and predictable |
| Long-term Maintainability | Depends on discipline | Strong by default |
| Typical Use Cases | Entities, mutable models, complex DTOs | DTOs, API responses, projections |








# JAVA Records
- [x] What Are Records? Need?
- [x] Basic Syntax & Example
- [x] Records Constructor 
- [x] Validations
- [x] Use Cases


A **transparent, immutable data carrier**



**Java team’s intent**

- Model _data_ explicitly
- Make immutability the default
- Enforce design at language level


### Canonical Constructor
- Default parameterized constructor
**Compact Constructor**

A **compact constructor** is a **shorter version of the canonical constructor**.



**Explicit Constructors**





## ✅ Use Cases of Java Records
1. **DTOs (Data Transfer Objects)**
    - API request & response models
    - Immutable, clean, no setters

2. **Request Payload Models**
    - `@RequestBody`  with validation
    - Constructor-based validation, safe inputs

3. **Response Models**
    - Read-only API responses
    - Clear service → controller contracts

4. **Event Objects (Kafka / Messaging)**
    - Domain events, integration events
    - Immutable, serializable payloads

5. **Cache Keys**
    - Map / cache identifiers
    - Safe `equals()`  and `hashCode()` 

6. **API Client Models**
    - Feign / RestTemplate responses
    - External service contracts

7. **Configuration Snapshots (Read-only)**
    - Derived or external config models
    - Thread-safe config data

8. **Stream & Functional Results**
    - Mapping, transformation outputs
    - Encourages immutability

9. **Pattern Matching & Deconstruction**
    - Modern Java `instanceof`  patterns
    - Cleaner data extraction

10. **Test Data Models**
    - Test fixtures & mocks
    - Simple, immutable test objects

---

## ❌ When NOT to Use Records
- JPA entities
- Mutable domain models
- Objects with identity or lifecycle
- Classes requiring inheritance
- Business-logic-heavy classes




<!--- Eraser file: https://app.eraser.io/workspace/vdXgGjnIc7O0pvycwfEg --->