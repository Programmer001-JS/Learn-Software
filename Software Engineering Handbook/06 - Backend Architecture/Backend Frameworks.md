# Backend Frameworks

> **In one line —** the code you did not have to write: routing, parsing, validation, sessions and database access, packaged so you can start on the actual product.

| | |
|---|---|
| **Category** | Overview note *(hub for this sub-section)* |
| **Architectural Layer** | Application |
| **Sub-topics** | [FastAPI](FastAPI.md) · [Express.js](Express.js.md) · [NestJS](NestJS.md) · [Spring Boot](Spring%20Boot.md) · [Laravel](Laravel.md) · [Django](Django.md) · [ASP.NET Core](ASP.NET%20Core.md) |
| **Related notes** | [Backend Fundamentals](Backend%20Fundamentals.md) · [Design Patterns](../07%20-%20Backend%20Design%20Patterns/Design%20Patterns.md) |

---

## 1. Short Definition

A backend framework provides the standard machinery every web application needs — routing, request parsing, middleware, validation, error handling — so that your code can be about your domain rather than about HTTP.

---

## 2. What every framework gives you

```text
Request
    ↓
Routing            which function handles this URL?
    ↓
Middleware         logging, CORS, auth, compression
    ↓
Parsing            body, query string, headers → typed objects
    ↓
Validation         reject bad input early
    ↓
Your handler       ← the only part that is actually your product
    ↓
Serialisation      objects → JSON
    ↓
Error handling     exceptions → status codes
    ↓
Response
```

---

## 3. The fundamental split

> [!IMPORTANT]
> The most useful way to compare frameworks is not by language but by **how much they decide for you.**

```text
MINIMAL / MICRO                      BATTERIES-INCLUDED
Express, Flask, Starlette            Django, Laravel, Spring Boot, Rails
    ↓                                     ↓
routing + middleware only            ORM, auth, admin, migrations, mail
you choose every library             conventions already decided
maximum flexibility                  maximum speed to first feature
consistency depends on discipline    consistency enforced by the framework
```

Neither is better. The question is whether you want to make those decisions, and whether your team will make them consistently.

---

## 4. Comparison

| Framework | Language | Style | Strongest at |
|---|---|---|---|
| **[FastAPI](FastAPI.md)** | Python | Micro + validation | Async APIs, automatic docs, AI/ML services |
| **[Django](Django.md)** | Python | Batteries-included | CRUD products, admin, fast delivery |
| **[Express.js](Express.js.md)** | JavaScript | Minimal | Small APIs, maximum flexibility |
| **[NestJS](NestJS.md)** | TypeScript | Structured, opinionated | Large TypeScript backends |
| **[Spring Boot](Spring%20Boot.md)** | Java | Batteries-included | Enterprise, high throughput, long-lived systems |
| **[Laravel](Laravel.md)** | PHP | Batteries-included | Rapid web applications, excellent ergonomics |
| **[ASP.NET Core](ASP.NET%20Core.md)** | C# | Batteries-included | Enterprise, strong typing, high performance |

---

## 5. How to choose

```text
1. What does the team already know?          ← usually decides it
    ↓
2. What does the ecosystem offer?            AI → Python. Enterprise → Java/.NET.
    ↓
3. Is the workload I/O-bound or CPU-bound?
    ↓
4. How long will this live, and who maintains it?
    ↓
5. Only then: performance benchmarks
```

> [!CAUTION]
> Framework benchmarks are close to irrelevant for most applications. A request spending 40 ms in the database and 2 ms in the framework does not get meaningfully faster with a framework that is twice as fast. Choose for ecosystem, team and maintainability.

---

## 6. What frameworks do not give you

- **Architecture** — a framework routes requests; it will not stop you putting all your logic in controllers
- **Domain modelling** — see [Domain Driven Design](../07%20-%20Backend%20Design%20Patterns/Domain%20Driven%20Design.md)
- **Security** — good defaults help, but authorisation logic is yours
- **Operational readiness** — logging, metrics, health checks, graceful shutdown

> [!IMPORTANT]
> Every framework's tutorial puts business logic in the controller. Every real codebase that followed that tutorial regrets it. See [Services](../07%20-%20Backend%20Design%20Patterns/Services.md) and [Clean Architecture](../07%20-%20Backend%20Design%20Patterns/Clean%20Architecture.md).

---

## 7. Security defaults worth knowing

Most frameworks protect you by default — until you disable it:

| Protection | Provided by |
|---|---|
| **SQL injection** | Parameterised queries in the ORM — bypassed by raw string SQL |
| **XSS** | Auto-escaping in templates — bypassed by "safe"/"raw" filters |
| **CSRF** | Token middleware — often disabled during debugging and never re-enabled |
| **Secure headers** | Middleware, sometimes opt-in |

> [!CAUTION]
> Nearly every real-world exploit of a mainstream framework involved someone turning a default off. Treat any `# nosec`, `|safe`, `csrf_exempt` or raw SQL as something requiring a written reason.

---

## 8. Mental Model

> [!NOTE]
> **A framework is a kit house.**
>
> The foundations, plumbing and wiring come already designed. A minimal framework gives you the foundation and lets you route the pipes; a batteries-included one hands you a finished house you must live in as designed. Both are far faster than pouring concrete yourself — and neither decides whether the rooms make sense.

---

## 9. Key Takeaway

> [!IMPORTANT]
> Frameworks solve HTTP, not architecture — choose one for its ecosystem and your team, then impose structure on it yourself.

---

## 10. Common Mistakes

- **Choosing on benchmarks** rather than ecosystem and team knowledge
- **Business logic in controllers**, making it untestable and unreusable
- **Disabling security defaults** and never restoring them
- **Fighting the framework's conventions** instead of following or replacing them wholesale
- **Adding a heavy framework** to a service that needed three endpoints

---

## 11. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 12. Workbook Exercise

- [ ] Take one controller in your project and list what in it is HTTP concern versus business logic.
- [ ] Find every place where a framework security default has been disabled, and justify each.
- [ ] Write down, in three sentences, why your current framework was chosen.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Web server
    ↓
FRAMEWORK  (routing, middleware, validation, serialisation)
    ↓
Your services and domain logic
    ↓
Repositories → database
```

## 2. Request Flow

```text
Input       an HTTP request
    ↓
Processing  routed, parsed, validated, dispatched to a handler
    ↓
Output      a serialised response, with errors mapped to status codes
```

## 3. Real-World Usage

**Instagram on Django** and **Netflix on Spring Boot** are both enormous systems built on batteries-included frameworks. In both cases the framework handled the mechanical parts and the teams supplied their own architecture on top.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Reusable machinery for handling HTTP and application plumbing |
| **Why does it exist?** | So every project does not rewrite routing, parsing and validation |
| **Where does it belong?** | Between the web server and your business logic |
| **When should I use it?** | Essentially always — chosen for ecosystem and team, not benchmarks |
