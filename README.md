# SpringBoot

## Q1- Why Do We Need @Bean When @Component Also Creates Beans? 🤔
Both `@Bean` and `@Component` create Spring-managed beans, but they are used in different scenarios.

| Feature           | @Bean                                                               | @Component                                                    |
|-------------------|---------------------------------------------------------------------|---------------------------------------------------------------|
| Usage             | Used inside `@Configuration` classes to manually create beans.      | Used to mark a class for automatic detection.                 |
| Control           | Full control over object creation and configuration.                | Spring creates the bean automatically.                        |
| External Classes? | Can be used for third-party classes (like REST clients, database connections). | Only works on classes you own and annotate.                   |
| Customization     | Allows modifying how the bean is created (e.g., constructor args, method calls). | Limited customization.                                        |

## Q2- How Does Spring Boot Find Beans?
a) Spring automatically scans packages and registers beans if they are annotated with:
- `@Component`
- `@Service`
- `@Repository`
- `@Controller`
- Explicit Bean Declaration (`@Bean`)

b) If a bean cannot be auto-detected (e.g., third-party classes), we register it manually using `@Bean`.

## Q3-Comparing Early vs Lazy Initialization
| Feature                     | Early Initialization (Default)                           | Lazy Initialization (`@Lazy`)                                  |
|-----------------------------|----------------------------------------------------------|---------------------------------------------------------------|
| When Bean is Created?       | At application startup                                   | Only when first used                                          |
| Memory Usage                | Higher (all beans preloaded)                             | Lower (only required beans loaded)                            |
| Startup Time                | Slower (loads all beans at once)                         | Faster (loads fewer beans at start)                           |
| Performance                 | Faster at runtime                                        | Might be slower on first use                                  |
| Use Case                    | Essential beans like Database Connections, Services      | Rarely used beans (e.g., large reports, email services)       |


## Q3-Bean Lifecycle Stages
A Spring Bean goes through the following steps:
![image](https://github.com/user-attachments/assets/9630d2dc-b1dc-414b-889d-b6d24e744a2f)

Stage	Description
1. Instantiation	Spring creates a bean instance.
2. Populate Properties	Dependencies (via @Autowired, constructor, setters) are injected.
3. Bean Post-Processing (Before Initialization)	BeanPostProcessor#postProcessBeforeInitialization() runs.
4. Initialization	If the bean implements InitializingBean or has @PostConstruct, it is executed.
5. Bean Post-Processing (After Initialization)	BeanPostProcessor#postProcessAfterInitialization() runs.
6. Bean is Ready to Use	The bean is fully initialized and available for use.
7. Destruction	If the bean implements DisposableBean or has @PreDestroy, cleanup happens before destruction.
   
## Q4-Dependency Injection in Spring Boot 
Ans- Dependency Injection (DI) is a design pattern where Spring Boot automatically provides the required dependencies (objects) instead of you manually creating them.

## Q5- How Spring Manages Dependency Injection?
@Component tells Spring to create and manage the object.
@Autowired tells Spring to inject the required dependency automatically.
Spring removes tight coupling and makes code more flexible and testable.
## Q6- Types of Dependency Injection in Spring Boot
A)Constructor Injection (Recommended)
@Autowired
public Car(Engine engine) {  // Injects dependency via constructor
    this.engine = engine;
}
B) Setter Injection
@Autowired
public void setEngine(Engine engine) {
    this.engine = engine;
}
C) Field Injection (Not Recommended)
@Autowired
private Engine engine;
## Why Constructor Injection is Best?
✅ Makes objects immutable
✅ Works well with unit testing
✅ Ensures all required dependencies are available at the time of object creation

🔹 Summary
Spring Boot automatically injects objects using @Autowired
Removes manual object creation (new)
Supports different injection types (Constructor, Setter, Field)
Helps in managing dependencies, making the code flexible & testable

## Circular dependency and how do we break circular dependency?

## Purpose and Use of API Key-Based Authorization in Spring Boot
API key-based authorization is a simple and effective way to secure REST APIs by requiring clients to include a valid API key in their requests. It ensures that only authorized users or applications can access certain endpoints.

## 📌 Purpose of API Key-Based Authorization
Authentication: Confirms the identity of the client making the request.
Access Control: Allows or denies access to specific endpoints based on the API key.
Security: Prevents unauthorized access, data breaches, and misuse of APIs.
Monitoring & Logging: Helps in tracking API usage and identifying suspicious activities.
Throttling & Rate Limiting: Restricts the number of requests from a particular API key to prevent abuse.



