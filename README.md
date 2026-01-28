# SpringBoot

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



























