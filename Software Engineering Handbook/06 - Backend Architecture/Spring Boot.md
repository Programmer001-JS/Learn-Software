# Spring Boot

> **In one line —** the Java enterprise standard: dependency injection, an enormous ecosystem, and sensible defaults that turned Spring from a configuration nightmare into something you can start in a minute.

| | |
|---|---|
| **Category** | Web Framework |
| **Architectural Layer** | Application |
| **Language** | Java / Kotlin |
| **Runs on** | [JVM](../03%20-%20Programming%20Languages%20and%20Runtime/JVM.md) |
| **Related notes** | [Backend Frameworks](Backend%20Frameworks.md) · [Dependency Injection](../07%20-%20Backend%20Design%20Patterns/Dependency%20Injection.md) · [Hibernate](../08%20-%20Databases%20and%20Data/Hibernate.md) · [Microservices](../10%20-%20Distributed%20Systems/Microservices.md) |

---

## 1. Short Definition

*What is it?*

Spring Boot is a framework that packages the Spring ecosystem with **auto-configuration** and sensible defaults. You add a dependency and the framework wires it up, instead of writing hundreds of lines of XML as classic Spring required.

---

## 2. Purpose

*What is its main purpose?*

To build long-lived, high-throughput backend services with strong typing, real multithreading, and the most mature enterprise ecosystem in existence.

---

## 3. Problem

*What engineering problem does it solve?*

Spring was powerful and famously painful to configure. Spring Boot's insight was **convention over configuration**: infer the setup from what is on the classpath.

```text
CLASSIC SPRING                     SPRING BOOT
XML config for everything          add spring-boot-starter-web
manual bean wiring                     ↓
weeks of setup                     an HTTP server, JSON, validation
                                   and DI are configured for you
```

---

## 4. Architecture Position

```text
Load balancer
    ↓
Embedded Tomcat / Netty          ← packaged inside the JAR
    ↓
┌──────────── SPRING BOOT ────────────┐
│  Filters / interceptors             │
│  @RestController   (HTTP layer)     │
│  @Service          (business logic) │
│  @Repository       (data access)    │
│  Spring Security   (auth)           │
└─────────────────┬───────────────────┘
                  ↓
        JPA / Hibernate → database
```

> [!TIP]
> Spring Boot applications embed their own web server and ship as a single executable JAR. There is no separate Tomcat to install and configure — a large part of why it works well in containers.

---

## 5. The core idea — dependency injection

```java
@RestController
public class UserController {
    private final UserService users;

    public UserController(UserService users) {   // ← injected by the container
        this.users = users;
    }

    @GetMapping("/users/{id}")
    public UserDto find(@PathVariable Long id) {
        return users.find(id);
    }
}
```

The container creates and wires every component. Swapping a real repository for a test double is configuration, not code surgery — see [Dependency Injection](../07%20-%20Backend%20Design%20Patterns/Dependency%20Injection.md).

---

## 6. What the ecosystem gives you

| Module | Provides |
|---|---|
| **Spring Web / WebFlux** | REST APIs, blocking or reactive |
| **Spring Data JPA** | Repositories generated from method names |
| **Spring Security** | Authentication, authorisation, OAuth2, JWT |
| **Spring Boot Actuator** | Health checks, metrics, readiness probes |
| **Spring Cloud** | Config, service discovery, circuit breakers |
| **Spring Batch** | Large-scale batch processing |

---

## 7. Real World Example

- **Netflix** built much of its microservice platform on Spring, and contributed Hystrix, Eureka and Ribbon back to it.
- **Banks and insurers** run enormous long-lived Spring systems — the ecosystem's stability and backwards compatibility are the selling point.
- **Kafka, Elasticsearch and Cassandra** are themselves JVM software, so Spring integrates with the data infrastructure naturally.

---

## 8. Input, Processing, Output

**Input:** an HTTP request arriving at the embedded server.
**Processing:** filters → interceptors → controller → service → repository, with the DI container supplying every dependency.
**Output:** a serialised response, with exceptions mapped by `@ControllerAdvice`.

---

## 9. Communication and Dependencies

- **JVM** — and all of its startup, memory and GC characteristics
- **Embedded Tomcat, Jetty or Netty**
- **JPA / [Hibernate](../08%20-%20Databases%20and%20Data/Hibernate.md)** for data access
- **Maven or Gradle** for builds
- **Actuator** for orchestrator health probes

---

## 10. Alternatives

```text
Spring Boot   the standard; largest ecosystem, heaviest startup
    ↓
Quarkus       built for containers and native compilation; very fast startup
    ↓
Micronaut     compile-time DI, no runtime reflection, small memory footprint
    ↓
Ktor          Kotlin-first, lightweight, coroutine-based
    ↓
ASP.NET Core  the closest equivalent outside the JVM
```

> [!TIP]
> **Quarkus and Micronaut exist specifically to fix Spring Boot's startup time and memory footprint** by moving work from runtime to compile time. If you are running many small JVM services or going serverless, they are worth evaluating.

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Use Spring Boot for long-running services where throughput, type safety and ecosystem maturity matter — enterprise systems, high-volume APIs, anything expected to live for a decade.

> [!CAUTION]
> - **Serverless and short-lived processes** — a 1–3 second cold start is disqualifying without GraalVM native compilation.
> - **Small services on constrained memory** — the JVM baseline is hundreds of megabytes.
> - **A team with no Java experience** — the learning curve is real, and "magic" auto-configuration is hard to debug without understanding what it replaced.

---

## 12. Advantages and Disadvantages

**Advantages**
- Exceptional throughput after JIT warm-up
- Real multithreading with no GIL
- The most complete enterprise ecosystem available
- Outstanding observability — Actuator, JFR, async-profiler
- Very strong backwards compatibility

**Disadvantages**
- Slow startup (1–3 seconds) and high memory baseline
- Auto-configuration is convenient until it does something unexpected
- Verbose compared with Python or TypeScript
- Large container images unless carefully trimmed
- The sheer size of the ecosystem is itself a learning barrier

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Throughput** | Very high once warmed up |
| **Startup** | 1–3 s; ~50 ms with GraalVM native images |
| **Memory** | 200–500 MB typical baseline |
| **Concurrency** | Thread pool; virtual threads (Java 21+) change this significantly |
| **Container** | 200–400 MB images |

> [!IMPORTANT]
> **Set `-Xmx` explicitly in containers.** A JVM that misreads the available memory will size its heap for the host rather than the container limit and get OOM-killed. Modern JVMs are container-aware, but explicit limits remain the safe practice.

---

## 14. Security Considerations

> [!CAUTION]
> **Log4Shell** was a Spring-ecosystem-adjacent catastrophe: a logged string containing `${jndi:...}` produced remote code execution across a large share of the Java world. Dependency management is a security responsibility, not a maintenance chore.

- **Spring Security is powerful and easy to misconfigure** — a permissive matcher silently opens endpoints
- **Java deserialisation of untrusted data** is a long-standing severe vulnerability class
- **Actuator endpoints expose internals** — restrict them; `/actuator/env` can leak configuration and secrets
- **Use `@PreAuthorize` on methods**, not only URL patterns, so authorisation survives refactoring
- Keep the JDK and every dependency patched; use `dependency-check` or Snyk in CI

---

## 15. Mental Model

> [!NOTE]
> **Spring Boot is a fully equipped industrial kitchen that takes twenty minutes to heat up.**
>
> Pointless for one sandwich. Once running, it serves hundreds of covers a night with consistency and equipment nothing lighter can match — and it has a tool for every job, which is both its strength and the reason the tour takes so long.

---

## 16. Mini Architecture Diagram

```text
Request
    ↓
Embedded Tomcat
    ↓
Security filter chain    → 401 / 403
    ↓
@RestController          (HTTP only)
    ↓
@Service                 (business logic, @Transactional)
    ↓
@Repository / JPA        (data access)
    ↓
Hibernate → PostgreSQL
    ↓
@ControllerAdvice        (exception → status code)
    ↓
Response
```

---

## 17. Complete Request Flow

```text
JVM starts, context built, beans wired      (~2 s, once)
    ↓
Request arrives → thread taken from the pool
    ↓
Spring Security filter chain authenticates and authorises
    ↓
Controller method invoked, arguments bound and validated
    ↓
Service method runs inside a @Transactional boundary
    ↓
Repository query → Hibernate → SQL → PostgreSQL
    ↓
Entity mapped to a DTO — never return entities directly
    ↓
JSON serialised, thread returned to the pool
    ↓
After a few thousand requests, hot paths are JIT-compiled
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> Spring Boot trades startup time and memory for throughput, real threading and the deepest enterprise ecosystem available — an excellent bargain for services that run for months and a poor one for anything short-lived.

---

## 19. Common Mistakes

- **Not setting `-Xmx`** in containers, causing OOM kills
- **Returning JPA entities from controllers**, leaking fields and causing lazy-loading errors
- **Exposing Actuator endpoints** publicly
- **Permissive Spring Security matchers** that open endpoints unintentionally
- **N+1 queries** from careless JPA relationships — see [Hibernate](../08%20-%20Databases%20and%20Data/Hibernate.md)
- **Benchmarking without warm-up**
- **Using it for serverless** without a native image

---

## 20. Open Source Technologies

- **Spring Boot**, **Spring Security**, **Spring Data**
- **Quarkus**, **Micronaut**, **Ktor** — lighter JVM alternatives
- **Hibernate**, **jOOQ** — data access
- **GraalVM** — native compilation for fast startup
- **Actuator + Micrometer + Prometheus** — metrics
- **async-profiler**, **JFR** — profiling

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Check whether your Actuator endpoints are reachable from outside the cluster.
- [ ] Measure request latency for the first 100 requests versus requests 10,000–10,100.
- [ ] Find one endpoint returning a JPA entity and replace it with a DTO.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Load balancer → embedded Tomcat → Spring Boot → JPA → database
```

## 2. Request Flow

```text
Input       an HTTP request
    ↓
Processing  security chain → controller → service → repository, all DI-wired
    ↓
Output      a DTO serialised to JSON
```

## 3. Real-World Usage

**Netflix** built its microservice platform on Spring and open-sourced much of it. The combination of JVM throughput and a deep ecosystem is why it remains the default for large-scale, long-lived backend systems.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | An auto-configuring, DI-based Java framework on the JVM |
| **Why does it exist?** | To make Spring's power usable without its configuration burden |
| **Where does it belong?** | Between an embedded server and your data layer |
| **When should I use it?** | Long-running, high-throughput services — not short-lived processes |
