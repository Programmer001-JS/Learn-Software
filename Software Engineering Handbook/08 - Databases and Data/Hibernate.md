# Hibernate

> **In one line —** the ORM that defined the category for Java, and taught an entire industry about lazy loading through production incidents.

| | |
|---|---|
| **Category** | ORM |
| **Architectural Layer** | Data access |
| **Language** | Java / Kotlin |
| **Standard** | The reference implementation of JPA |
| **Related notes** | [ORM](ORM.md) · [Spring Boot](../06%20-%20Backend%20Architecture/Spring%20Boot.md) · [SQL](SQL.md) · [Transactions](Transactions.md) · [JVM](../03%20-%20Programming%20Languages%20and%20Runtime/JVM.md) |

---

## 1. Short Definition

*What is it?*

Hibernate maps Java classes to database tables. It is the most widely used implementation of **JPA** (Jakarta Persistence API), the Java persistence standard, and is what Spring Data JPA runs on underneath.

---

## 2. Purpose

*What is its main purpose?*

To let Java applications work with objects and relationships rather than result sets, while managing transactions, caching and change tracking automatically.

---

## 3. Problem

*What engineering problem does it solve?*

Java's JDBC is extremely verbose: connection handling, statement preparation, result-set iteration and manual mapping for every query. Hibernate removed that boilerplate and became the default for enterprise Java.

---

## 4. Architecture Position

```text
Spring Boot service
    ↓
Spring Data JPA repository
    ↓
JPA API  (the standard)
    ↓
HIBERNATE  (the implementation)
    ↓
JDBC driver
    ↓
Database
```

---

## 5. The Persistence Context — the concept everything depends on

```text
The Persistence Context is:
    a first-level CACHE       — the same entity fetched twice is the same object
    a CHANGE TRACKER          — modified entities are flushed automatically
    a transaction-scoped unit of work
```

```java
@Transactional
public void rename(Long id, String name) {
    Order order = repo.findById(id).orElseThrow();
    order.setName(name);            // no explicit save() needed
}                                   // ← flush + commit happen here
```

> [!IMPORTANT]
> **Dirty checking** surprises newcomers constantly: modifying a *managed* entity inside a transaction issues an `UPDATE` even with no `save()` call. Convenient once understood, and a source of accidental writes until then.

---

## 6. Lazy loading and its two famous failures

**N+1 queries:**

```java
List<Order> orders = repo.findAll();          // 1 query
for (Order o : orders) o.getCustomer().getName();   // N queries
```

```java
@Query("SELECT o FROM Order o JOIN FETCH o.customer")   // ✓ 1 query
```

**LazyInitializationException:**

```text
Entity loaded inside a transaction
    ↓
Transaction ends, persistence context closed
    ↓
Controller serialises the entity → touches a lazy relation
    ↓
LazyInitializationException — no session available
```

> [!CAUTION]
> This exception is the single most common Hibernate error, and the fix is not "open the session longer". It is **never returning entities from controllers**. Map to a DTO inside the transaction, and the problem cannot occur — while also preventing the serialisation of fields you never intended to expose.

---

## 7. Fetch types

```text
@ManyToOne   default = EAGER   ← usually wrong; makes every load a join
@OneToMany   default = LAZY    ← correct default

Recommended: mark everything LAZY, then fetch explicitly per query
             with JOIN FETCH or an EntityGraph.
```

---

## 8. Caching

```text
FIRST-LEVEL    persistence context, per transaction    always on
SECOND-LEVEL   shared across sessions (Ehcache, Redis) optional
QUERY CACHE    caches query results                    use with care
```

> [!CAUTION]
> The second-level cache is powerful and easy to misuse. If any other application or a manual SQL statement writes to the same tables, Hibernate's cache does not know, and it will serve stale data indefinitely.

---

## 9. Real World Example

- **Spring Data JPA** — the standard data layer for the overwhelming majority of Spring Boot applications.
- **Enterprise systems** running for a decade or more, where JPA portability across databases genuinely mattered.
- **The N+1 and LazyInitializationException problems** are so common that they are effectively a Java rite of passage.

---

## 10. Communication and Dependencies

- **A JDBC driver** and a connection pool (HikariCP by default in Spring Boot)
- **JPA annotations** on entity classes
- **Flyway or Liquibase** for migrations — `hbm2ddl.auto` is not a migration strategy
- **Spring's transaction management**, usually via `@Transactional`

---

## 11. Alternatives

```text
Hibernate / JPA   full ORM, dominant, heavyweight
    ↓
jOOQ              typed SQL DSL — SQL-first, excellent for complex queries
    ↓
Spring JDBC       thin JDBC wrapper, you write the SQL
    ↓
MyBatis           SQL in XML or annotations, mapped to objects
    ↓
Exposed / Ktorm   Kotlin-native alternatives
```

> [!TIP]
> **jOOQ is the strongest alternative** when queries are the hard part. Many teams run Hibernate for CRUD and jOOQ for reporting, in the same application.

---

## 12. When To Use / When NOT To Use

> [!TIP]
> Use Hibernate for standard entity CRUD in Java applications, especially within Spring Boot where the integration is seamless.

> [!CAUTION]
> Avoid it for complex reporting, heavy aggregation and bulk operations. Generated HQL for a five-table report is usually worse than the SQL you would write, and bulk updates through entities are dramatically slower than a single statement.

---

## 13. Advantages and Disadvantages

**Advantages**
- Removes enormous JDBC boilerplate
- Automatic dirty checking and transaction integration
- A standard (JPA), so knowledge transfers across projects
- Mature, extremely well documented, huge community
- Caching layers built in

**Disadvantages**
- Steep learning curve — the persistence context is genuinely subtle
- N+1 and LazyInitializationException are near-universal experiences
- Generated SQL for complex queries is often poor
- Bulk operations are slow through entities
- The abstraction leaks constantly, so you must know SQL anyway

---

## 14. Performance Impact

| Aspect | Impact |
|---|---|
| **N+1** | The dominant risk, as with every ORM |
| **EAGER `@ManyToOne`** | Silently joins on every load |
| **Bulk operations** | Use `@Modifying` queries or JDBC batch, not entity loops |
| **First-level cache** | Free deduplication within a transaction |
| **Open Session In View** | Convenient, and holds a connection for the whole request — disable it |

> [!CAUTION]
> **Spring Boot enables `spring.jpa.open-in-view` by default.** It keeps the persistence context open through view rendering, which hides `LazyInitializationException` and holds a database connection far longer than necessary. Turning it off surfaces real problems early and is standard advice for production systems.

---

## 15. Security Considerations

> [!IMPORTANT]
> JPQL and criteria queries are parameterised. Injection requires string concatenation:

```java
"SELECT u FROM User u WHERE u.email = '" + email + "'"   // ✗ injectable
"SELECT u FROM User u WHERE u.email = :email"            // ✓ bound parameter
```

- **Never return entities from REST controllers** — they serialise every field, including relationships and anything added later
- **Java deserialisation of untrusted data** remains a severe vulnerability class in this ecosystem
- **`hbm2ddl.auto=update` in production** lets the application modify the schema — it should never have that right
- **Least privilege**: the runtime database user should not hold DDL permissions

---

## 16. Mental Model

> [!NOTE]
> **Hibernate is an assistant who quietly writes down everything you change and files it when you leave the room.**
>
> Extremely convenient once you know it is happening. Deeply confusing the first time a document gets updated because you edited an object and never asked for it to be saved.

---

## 17. Mini Architecture Diagram

```text
Service (@Transactional)
    ↓
Repository (Spring Data JPA)
    ↓
Persistence context  — first-level cache, dirty checking
    ↓
Hibernate → SQL
    ↓
HikariCP pool → JDBC → database
```

---

## 18. Complete Request Flow

```text
Controller calls a @Transactional service method
    ↓
Transaction begins; persistence context created
    ↓
repo.findById(42) → SELECT; entity becomes MANAGED
    ↓
Business logic mutates the entity — no SQL yet
    ↓
A lazy relation is accessed → extra SELECT (N+1 risk lives here)
    ↓
Method returns → flush: dirty entities produce UPDATE statements
    ↓
COMMIT; persistence context closed
    ↓
Controller maps to a DTO — inside the transaction, or the lazy relations
are already unreachable
```

---

## 19. Key Takeaway

> [!IMPORTANT]
> Hibernate tracks entity changes automatically inside a transaction — and its two universal failures, N+1 and LazyInitializationException, are both solved by explicit fetching and by never returning entities from controllers.

---

## 20. Common Mistakes

- **Returning entities from controllers** — the root cause of most Hibernate pain
- **N+1 queries** from default lazy relations in loops
- **`@ManyToOne` left EAGER**
- **Leaving `open-in-view` enabled** in production
- **`hbm2ddl.auto=update`** instead of Flyway or Liquibase
- **Bulk updates entity by entity**
- **Second-level cache** on tables written by other systems

---

## 21. Open Source Technologies

- **Hibernate ORM**, **Spring Data JPA**
- **jOOQ**, **MyBatis**, **Spring JDBC** — alternatives
- **Flyway**, **Liquibase** — migrations
- **HikariCP** — the connection pool
- **hibernate-statistics**, **datasource-proxy**, **p6spy** — see the generated SQL and query counts

---

## 22. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 23. Workbook Exercise

- [ ] Check whether `spring.jpa.open-in-view` is enabled in your application, and what breaks when you disable it.
- [ ] Enable SQL logging and count the queries for your heaviest endpoint.
- [ ] Find one controller returning an entity and replace it with a DTO.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Service → Spring Data JPA → Hibernate → JDBC → database
```

## 2. Request Flow

```text
Input       repository calls inside a transaction
    ↓
Processing  entities managed in a persistence context; changes tracked and flushed
    ↓
Output      persisted state, plus DTOs mapped before the context closes
```

## 3. Real-World Usage

**Spring Data JPA on Hibernate** is the default data layer across enterprise Java. Its two classic failure modes are so widely encountered that recognising them instantly is a genuine marker of Java backend experience.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | The reference JPA implementation, mapping Java objects to tables |
| **Why does it exist?** | To remove JDBC boilerplate and manage persistence automatically |
| **Where does it belong?** | Between Spring services and the JDBC driver |
| **When should I use it?** | Entity CRUD in Java — with jOOQ or SQL for reporting |
