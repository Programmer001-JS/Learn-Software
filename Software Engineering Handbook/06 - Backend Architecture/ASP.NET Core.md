# ASP.NET Core

> **In one line —** Microsoft's cross-platform web framework: strongly typed, genuinely fast, with async built into the language rather than bolted on.

| | |
|---|---|
| **Category** | Web Framework |
| **Architectural Layer** | Application |
| **Language** | C# / F# |
| **Runs on** | [.NET Runtime](../03%20-%20Programming%20Languages%20and%20Runtime/dotNET%20Runtime.md) |
| **Related notes** | [Backend Frameworks](Backend%20Frameworks.md) · [Spring Boot](Spring%20Boot.md) · [Entity Framework](../08%20-%20Databases%20and%20Data/Entity%20Framework.md) · [Dependency Injection](../07%20-%20Backend%20Design%20Patterns/Dependency%20Injection.md) |

---

## 1. Short Definition

*What is it?*

ASP.NET Core is a cross-platform framework for building web applications and APIs in C#. It includes a built-in dependency injection container, a middleware pipeline, and **Kestrel** — one of the fastest managed HTTP servers available.

---

## 2. Purpose

*What is its main purpose?*

To build high-throughput, strongly typed backend services that run on Linux containers as comfortably as on Windows.

---

## 3. Problem

*What engineering problem does it solve?*

Classic ASP.NET was Windows-only, tied to IIS, and heavy. ASP.NET Core was a full rewrite: cross-platform, open source, modular and considerably faster — which turned .NET from an enterprise-Windows choice into a general server platform.

---

## 4. Architecture Position

```text
Nginx / cloud load balancer   (optional — Kestrel can face the internet)
    ↓
Kestrel                        ← the built-in HTTP server
    ↓
┌──────── ASP.NET CORE ────────┐
│  Middleware pipeline          │
│  Routing                      │
│  Controller / Minimal API     │
│  Model binding + validation   │
│  Service layer  (DI)          │
│  EF Core                      │
└──────────────┬────────────────┘
               ↓
        SQL Server / PostgreSQL
```

---

## 5. Two programming models

```csharp
// Minimal API — for small services
app.MapGet("/users/{id}", async (int id, IUserService svc) =>
    await svc.FindAsync(id) is User u ? Results.Ok(u) : Results.NotFound());

// Controllers — for larger applications
[ApiController]
[Route("users")]
public class UsersController : ControllerBase
{
    private readonly IUserService _svc;
    public UsersController(IUserService svc) => _svc = svc;   // ← injected

    [HttpGet("{id}")]
    public async Task<ActionResult<UserDto>> Get(int id) => await _svc.FindAsync(id);
}
```

> [!TIP]
> **Minimal APIs** suit microservices and small endpoints; **controllers** suit larger applications where filters, conventions and grouping pay off. Both use the same DI container and middleware pipeline.

---

## 6. Async is a language feature

```csharp
var user = await _db.Users.FindAsync(id);   // thread RELEASED while waiting
```

The compiler rewrites the method into a state machine. The thread returns to the pool during I/O rather than blocking, which is why ASP.NET Core sustains very high concurrency with a small thread pool.

> [!CAUTION]
> **Never call `.Result` or `.Wait()` on a Task in a request path.** It blocks a pool thread waiting for work that needs a pool thread — the classic cause of thread-pool starvation and deadlocks under load.

---

## 7. Real World Example

- **Stack Overflow** served enormous traffic from a handful of .NET servers — the reference case for a well-tuned monolith.
- **Microsoft's own services**, Bing and much of Azure.
- **Enterprise line-of-business systems** — the largest single .NET segment.
- **TechEmpower benchmarks** consistently place ASP.NET Core among the fastest full-featured frameworks.

---

## 8. Communication and Dependencies

- **Kestrel** — the HTTP server, embedded in the application
- **[Entity Framework Core](../08%20-%20Databases%20and%20Data/Entity%20Framework.md)** or Dapper for data access
- **Built-in DI** — no third-party container needed
- **`IOptions` configuration** from JSON, environment variables and secret stores
- **Serilog / OpenTelemetry** for logging and tracing

---

## 9. Alternatives

```text
ASP.NET Core   fast, typed, cross-platform, excellent async
    ↓
Spring Boot    the closest equivalent; larger ecosystem, slower startup
    ↓
NestJS         similar structure in TypeScript
    ↓
Go net/http    smaller binaries, simpler runtime, less framework
    ↓
FastAPI        far lighter, but Python concurrency constraints
```

---

## 10. When To Use / When NOT To Use

> [!TIP]
> Use ASP.NET Core for long-running, high-throughput services needing strong typing and real parallelism — especially with existing C# expertise or Azure integration.

> [!CAUTION]
> - **Data science and AI** — the ecosystem is Python's, and .NET is not close.
> - **Very small serverless functions** — use Native AOT, or a lighter runtime.
> - **Teams with no C# experience** and no other reason to adopt it.

---

## 11. Advantages and Disadvantages

**Advantages**
- Among the fastest managed-runtime frameworks
- Real multithreading with no GIL
- `async`/`await` designed into the language
- DI, configuration, logging and health checks built in
- Excellent tooling — Visual Studio, Rider, built-in diagnostics
- Faster startup than the JVM, with Native AOT available

**Disadvantages**
- Smaller open-source ecosystem than the JVM or Python in several domains
- Rapid version cadence; major releases every year
- Historical Windows association still shapes hiring
- EF Core makes inefficient queries easy to write
- GC pauses affect tail latency, as in any managed runtime

---

## 12. Performance Impact

| Aspect | Impact |
|---|---|
| **Throughput** | Very high — frequently top-tier in independent benchmarks |
| **Startup** | ~100 ms; ~10 ms with Native AOT |
| **Memory** | Lower than the JVM, higher than Go |
| **Concurrency** | Real threads plus async — excellent for both CPU and I/O |

---

## 13. Security Considerations

> [!CAUTION]
> **Mass assignment** — binding a request body directly to an EF entity lets a caller set any property, including `IsAdmin`. Bind to a DTO, never to an entity.

- **`[Authorize]` is authentication plus role checks** — object ownership must be checked in your code
- **`BinaryFormatter`** is obsolete and dangerous; do not deserialise untrusted data
- **Developer exception pages must never run in production** — check `ASPNETCORE_ENVIRONMENT`
- **User secrets are for development only**; use Key Vault or a secrets manager in production
- **Enable HSTS and HTTPS redirection**; both are one line in the pipeline
- Keep the runtime patched — Microsoft ships regular security releases

---

## 14. Mental Model

> [!NOTE]
> **ASP.NET Core is Spring Boot's sibling raised in a different house.**
>
> Same architecture — DI container, middleware pipeline, ORM, strong typing — with a lighter runtime, faster startup, and an asynchronous model built into the language rather than added later.

---

## 15. Mini Architecture Diagram

```text
Request
    ↓
Kestrel
    ↓
Middleware: HTTPS redirect → HSTS → auth → authorisation → routing
    ↓
Controller / Minimal API endpoint
    ↓
Model binding + validation  → 400
    ↓
Service (injected)
    ↓
EF Core → database
    ↓
DTO → JSON
    ↓
Exception handler middleware → status code
```

---

## 16. Complete Request Flow

```text
Runtime starts, DI container built             (~100 ms, once)
    ↓
Request arrives at Kestrel
    ↓
Middleware pipeline runs in registration order
    ↓
Authentication → claims principal; Authorization → 403 if refused
    ↓
Route matched; controller resolved from DI
    ↓
Model binding + DataAnnotations validation
    ↓
await service call → THREAD RETURNED TO THE POOL
    ↓
Other requests use that thread meanwhile
    ↓
EF Core query completes → continuation scheduled
    ↓
Entity mapped to a DTO, serialised, returned
```

---

## 17. Key Takeaway

> [!IMPORTANT]
> ASP.NET Core combines high throughput, real threading and language-level async — and its most common security mistake is binding requests directly to database entities.

---

## 18. Common Mistakes

- **`.Result` / `.Wait()`** in request paths, starving the thread pool
- **Binding to EF entities** instead of DTOs — mass assignment
- **Returning entities** from controllers, leaking fields and causing lazy-loading issues
- **Developer exception page in production**
- **Middleware in the wrong order** — authorisation before authentication does nothing
- **N+1 queries** in EF Core
- **Ignoring container memory limits**, misleading the garbage collector

---

## 19. Open Source Technologies

- **ASP.NET Core**, **.NET runtime** — fully open source
- **Entity Framework Core**, **Dapper** — data access
- **Serilog**, **OpenTelemetry** — logging and tracing
- **FluentValidation** — validation beyond attributes
- **xUnit**, **Testcontainers** — testing
- **dotnet-counters**, **dotnet-trace** — diagnostics

---

## 20. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 21. Workbook Exercise

- [ ] Search your codebase for `.Result` and `.Wait()` in request paths.
- [ ] Find one endpoint binding directly to an EF entity and introduce a DTO.
- [ ] Check your middleware order — authentication must come before authorisation.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Load balancer → Kestrel → middleware → controller → service → EF Core → database
```

## 2. Request Flow

```text
Input       an HTTP request
    ↓
Processing  middleware pipeline, DI-resolved controller, async service calls
    ↓
Output      a DTO serialised to JSON
```

## 3. Real-World Usage

**Stack Overflow** ran one of the busiest sites on the web from a small number of .NET servers. It remains the clearest demonstration that a fast runtime and a well-designed monolith can outperform far more complex distributed architectures.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A cross-platform, strongly typed, high-performance web framework for C# |
| **Why does it exist?** | To make .NET a first-class platform outside Windows |
| **Where does it belong?** | Between Kestrel and your data layer |
| **When should I use it?** | High-throughput typed services — not data science or tiny functions |
