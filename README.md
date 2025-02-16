# SpringBoot

# SpringBoot

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

## Comparing Early vs Lazy Initialization
| Feature                     | Early Initialization (Default)                           | Lazy Initialization (`@Lazy`)                                  |
|-----------------------------|----------------------------------------------------------|---------------------------------------------------------------|
| When Bean is Created?       | At application startup                                   | Only when first used                                          |
| Memory Usage                | Higher (all beans preloaded)                             | Lower (only required beans loaded)                            |
| Startup Time                | Slower (loads all beans at once)                         | Faster (loads fewer beans at start)                           |
| Performance                 | Faster at runtime                                        | Might be slower on first use                                  |
| Use Case                    | Essential beans like Database Connections, Services      | Rarely used beans (e.g., large reports, email services)       |
