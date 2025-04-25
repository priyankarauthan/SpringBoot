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































