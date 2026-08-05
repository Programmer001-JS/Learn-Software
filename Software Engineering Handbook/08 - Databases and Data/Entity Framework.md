# Entity Framework Core

> **In one line —** Microsoft's ORM for .NET: LINQ queries compiled into SQL, with the same N+1 traps as every ORM and better defaults than most.

| | |
|---|---|
| **Category** | ORM |
| **Architectural Layer** | Data access |
| **Language** | C# / F# |
| **Abbreviation** | EF Core |
| **Related notes** | [ORM](ORM.md) · [ASP.NET Core](../06%20-%20Backend%20Architecture/ASP.NET%20Core.md) · [SQL](SQL.md) · [Transactions](Transactions.md) · [.NET Runtime](../03%20-%20Programming%20Languages%20and%20Runtime/.NET%20Runtime.md) |

---

## 1. Short Definition

*What is it?*

EF Core maps C# classes to database tables and translates **LINQ** expressions into SQL. Queries are written in C#, checked by the compiler, and executed as parameterised SQL.

---

## 2. Purpose

*What is its main purpose?*

To give .NET applications strongly typed, compiler-checked data access, with change tracking and migrations included.

---

## 3. Problem

*What engineering problem does it solve?*

```text
WITHOUT                               WITH EF CORE
SqlCommand, SqlDataReader             context.Orders
manual parameter binding                  .Where(o => o.Total > 100)
manual column-to-property mapping         .Include(o => o.Customer)
string SQL, no compile checking           .ToListAsync();

typo in a column name = runtime error ↑ typo = compile error
```

---

## 4. Architecture Position

```text
ASP.NET Core controller
    ↓
Service layer
    ↓
DbContext          ← unit of work + change tracker + identity map
    ↓
EF Core query translation (LINQ → SQL)
    ↓
ADO.NET provider
    ↓
SQL Server / PostgreSQL / SQLite
```

---

## 5. The DbContext

```csharp
public class AppDb : DbContext
{
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<User>  Users  => Set<User>();
}
```

```text
The DbContext is:
    a TRANSACTION boundary
    a CHANGE TRACKER   — modified entities produce UPDATEs on SaveChanges
    an IDENTITY MAP    — the same row is the same object instance
```

> [!IMPORTANT]
> **DbContext is not thread-safe and must be short-lived.** Register it as **scoped** (per request) — never as a singleton. A singleton DbContext accumulates tracked entities forever, leaks memory, and serves one user's cached data to another.

---

## 6. Client-side evaluation — the trap that used to be silent

```csharp
// ✗ MyHelper cannot be translated to SQL
context.Orders.Where(o => MyHelper.IsValid(o)).ToList();
```

In EF Core 2.x this silently fetched the **entire table** into memory and filtered it in C#. Since EF Core 3.0 it throws an exception instead.

> [!CAUTION]
> If you maintain an older EF Core application, this is worth auditing specifically. A query that "works" may be loading a million rows to return ten.

---

## 7. Loading strategies

```csharp
// ✗ N+1 — lazy loading, if enabled
foreach (var o in context.Orders) { var n = o.Customer.Name; }

// ✓ eager loading
context.Orders.Include(o => o.Customer).ThenInclude(c => c.Address)

// ✓ projection — fetches only what is needed, no tracking
context.Orders.Select(o => new OrderDto(o.Id, o.Customer.Name))
```

> [!TIP]
> **Lazy loading is off by default in EF Core**, which is a genuinely good decision — the most common ORM bug is opt-in rather than opt-out. Keep it off.

---

## 8. Tracking vs no-tracking

```csharp
context.Orders.ToList()                 // tracked — change detection overhead
context.Orders.AsNoTracking().ToList()  // read-only, measurably faster
```

> [!TIP]
> Use `AsNoTracking()` for every read-only query. On large result sets the difference is substantial, and forgetting it is one of the most common EF Core performance issues.

---

## 9. Migrations

```bash
dotnet ef migrations add AddOrderStatus
dotnet ef database update
```

Migrations are generated as C# classes — reviewable, editable and version-controlled.

> [!CAUTION]
> Never use `EnsureCreated()` in production; it bypasses migrations entirely. And always read a generated migration before applying it — auto-generated DDL has dropped columns and rebuilt large tables in production more than once.

---

## 10. Real World Example

- **The default data layer for ASP.NET Core** applications.
- **Enterprise line-of-business systems** — EF Core plus SQL Server is the archetypal Microsoft stack.
- **Dapper alongside EF Core** is a common hybrid: EF for CRUD, Dapper for hand-written reporting queries.

---

## 11. Communication and Dependencies

- **An ADO.NET provider** — SQL Server, Npgsql for PostgreSQL, SQLite
- **The DI container** — DbContext registered as scoped
- **Migrations tooling** — `dotnet ef`
- **Connection pooling** — handled by the provider

---

## 12. Alternatives

```text
EF Core     full ORM, LINQ, migrations, change tracking
    ↓
Dapper      micro-ORM: you write the SQL, it maps the results — very fast
    ↓
Raw ADO.NET  full control, full boilerplate
    ↓
Linq2Db     LINQ-based, closer to SQL, strong bulk support
```

> [!TIP]
> **Dapper is the standard companion, not a competitor.** Use EF Core for entity CRUD and Dapper for the reporting queries where you want to write the SQL yourself.

---

## 13. When To Use / When NOT To Use

> [!TIP]
> Use EF Core for typical CRUD in .NET applications where compile-time checked queries and integrated migrations are valuable.

> [!CAUTION]
> Avoid it for complex reporting, heavy aggregation and bulk operations. `ExecuteUpdate`/`ExecuteDelete` (EF Core 7+) help considerably, but for large batch work raw SQL or Dapper remains better.

---

## 14. Advantages and Disadvantages

**Advantages**
- LINQ gives compile-time checked queries
- Lazy loading off by default — a better default than most ORMs
- Excellent, reviewable migrations
- Strong integration with ASP.NET Core DI
- Provider model supports several databases

**Disadvantages**
- Generated SQL for complex LINQ can be poor
- Change tracking costs on large result sets
- DbContext lifetime mistakes are a common production bug
- Bulk operations were historically weak
- The abstraction leaks — SQL knowledge is still required

---

## 15. Performance Impact

| Aspect | Impact |
|---|---|
| **Tracking** | Real overhead; use `AsNoTracking()` for reads |
| **N+1** | Possible with lazy loading or unprojected loops |
| **`Include` chains** | Can produce enormous cartesian joins — use `AsSplitQuery()` |
| **Bulk operations** | `ExecuteUpdate`/`ExecuteDelete`, or Dapper |
| **Projections** | `Select` into a DTO fetches fewer columns and skips tracking |

---

## 16. Security Considerations

> [!IMPORTANT]
> LINQ queries are always parameterised — EF Core does not concatenate values into SQL. Injection requires raw string SQL:

```csharp
context.Users.FromSqlRaw($"SELECT * FROM Users WHERE Email = '{email}'")   // ✗
context.Users.FromSqlInterpolated($"SELECT * FROM Users WHERE Email = {email}")  // ✓ parameterised
```

- **Mass assignment** — binding a request body directly to an entity lets a caller set `IsAdmin`. Bind to a DTO
- **Never return entities from controllers** — they serialise every property and can trigger lazy loads mid-serialisation
- **Global query filters** are an excellent multi-tenancy defence: `HasQueryFilter(e => e.TenantId == _tenant)` applies to every query automatically
- **Least privilege** — the runtime user should not hold DDL rights; migrations run under a separate account

---

## 17. Mental Model

> [!NOTE]
> **EF Core is a bilingual assistant who writes your SQL for you.**
>
> You describe the query in C# and they produce correct, parameterised SQL. For everyday requests it is faster and safer than writing it yourself. For a complicated report, you want to see the SQL — because their translation is grammatical and occasionally very inefficient.

---

## 18. Mini Architecture Diagram

```text
Controller
    ↓
Service
    ↓
DbContext (scoped) — change tracker, identity map
    ↓
LINQ → SQL translation
    ↓
ADO.NET provider → pool
    ↓
Database
```

---

## 19. Complete Request Flow

```text
Request → scoped DbContext created by DI
    ↓
LINQ query written in C#, compile-time checked
    ↓
Translated to parameterised SQL
    ↓
AsNoTracking()?  → no change tracking overhead
Include()?       → eager loading, single or split query
    ↓
Rows returned and materialised into entities
    ↓
Mutations tracked; SaveChangesAsync() issues UPDATE/INSERT in one transaction
    ↓
DbContext disposed at the end of the request; connection returned to the pool
    ↓
Response built from a DTO, never from the entity
```

---

## 20. Key Takeaway

> [!IMPORTANT]
> EF Core gives compile-time checked queries and sane defaults — keep the DbContext scoped per request, use `AsNoTracking()` for reads, and project into DTOs rather than returning entities.

---

## 21. Common Mistakes

- **Singleton or long-lived DbContext** — memory growth and cross-request data leaks
- **Missing `AsNoTracking()`** on read-only queries
- **Returning entities from controllers**
- **Binding request bodies to entities** — mass assignment
- **Huge `Include` chains** producing cartesian explosions
- **`EnsureCreated()`** instead of migrations
- **Applying generated migrations without reading them**

---

## 22. Open Source Technologies

- **EF Core** — fully open source
- **Dapper** — the standard micro-ORM companion
- **Npgsql** — the PostgreSQL provider
- **EFCore.BulkExtensions** — high-performance bulk operations
- **MiniProfiler** — see the SQL each request generates

---

## 23. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 24. Workbook Exercise

- [ ] Check your DI registration — is DbContext scoped?
- [ ] Add `AsNoTracking()` to your heaviest read query and measure the difference.
- [ ] Log the SQL for one `Include`-heavy query and check whether it needs `AsSplitQuery()`.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Controller → Service → DbContext → SQL → provider → database
```

## 2. Request Flow

```text
Input       a LINQ query or entity mutation
    ↓
Processing  translated to parameterised SQL; changes tracked until SaveChanges
    ↓
Output      materialised entities, and a single transactional write
```

## 3. Real-World Usage

**EF Core with Dapper alongside it** is the common production pattern in .NET: the ORM handles entity CRUD, and hand-written SQL handles the reports where LINQ translation would be both slower and less readable.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Microsoft's ORM translating LINQ into SQL |
| **Why does it exist?** | To give .NET typed, compiler-checked data access |
| **Where does it belong?** | Between your service layer and the database provider |
| **When should I use it?** | Entity CRUD in .NET — with Dapper or SQL for reporting |
