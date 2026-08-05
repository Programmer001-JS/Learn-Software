# NestJS

> **In one line —** Angular's architecture applied to the backend: modules, dependency injection and decorators, so a large TypeScript codebase has one shape instead of twenty.

| | |
|---|---|
| **Category** | Web Framework |
| **Architectural Layer** | Application |
| **Language** | TypeScript (Node.js) |
| **Runs on** | Express or Fastify underneath |
| **Related notes** | [Express.js](Express.js.md) · [Dependency Injection](../07%20-%20Backend%20Design%20Patterns/Dependency%20Injection.md) · [Clean Architecture](../07%20-%20Backend%20Design%20Patterns/Clean%20Architecture.md) · [TypeScript](../05%20-%20Frontend%20Architecture/TypeScript.md) |

---

## 1. Short Definition

*What is it?*

NestJS is an opinionated TypeScript backend framework built around **modules**, **providers** and **dependency injection**. It sits on top of [Express](Express.js.md) (or Fastify) and adds the structure they deliberately omit.

---

## 2. Purpose

*What is its main purpose?*

To give large Node backends a consistent architecture that survives team growth — so that any developer opening any module finds the same layout.

---

## 3. Problem

*What engineering problem does it solve?*

Express projects have no prescribed structure. With three developers this is freedom; with thirty it means every feature is organised differently and nothing can be moved safely.

```text
EXPRESS                          NESTJS
routes/  controllers/            src/users/
  or handlers/  or api/            users.module.ts
  — different in every project     users.controller.ts
                                   users.service.ts
                                   users.repository.ts
                                 ← identical for every feature
```

---

## 4. Architecture Position

```text
HTTP request
    ↓
Express / Fastify adapter
    ↓
┌──────────────── NESTJS ────────────────┐
│  Middleware                            │
│  Guards          (authorisation)       │
│  Interceptors    (before/after logic)  │
│  Pipes           (validation)          │
│  Controller      (HTTP layer only)     │
│  Service         (business logic)      │
│  Repository      (data access)         │
└────────────────────┬───────────────────┘
                     ↓
                 Database
```

---

## 5. The building blocks

```typescript
@Controller('users')
export class UsersController {
  constructor(private readonly users: UsersService) {}   // ← injected

  @Get(':id')
  @UseGuards(AuthGuard)
  findOne(@Param('id', ParseIntPipe) id: number) {
    return this.users.findOne(id);
  }
}
```

| Concept | Role |
|---|---|
| **Module** | Groups a feature and declares its dependencies |
| **Controller** | HTTP only — routes, params, status codes |
| **Service (provider)** | Business logic; injectable and testable |
| **Guard** | Decides whether a request may proceed (authorisation) |
| **Pipe** | Transforms and validates input |
| **Interceptor** | Wraps handlers — logging, caching, response shaping |

> [!IMPORTANT]
> The controller/service split is enforced by the framework's shape rather than by discipline. That is the main practical benefit: business logic ends up somewhere testable by default rather than by good intentions.

---

## 6. Dependency injection

Nest has a real IoC container: providers are registered in a module and injected by type. Swapping a real repository for a fake in tests is a one-line override, with no monkey-patching.

See [Dependency Injection](../07%20-%20Backend%20Design%20Patterns/Dependency%20Injection.md).

---

## 7. Real World Example

- **Large TypeScript backends** where a team of ten or more needs a shared structure.
- **Full-stack TypeScript** — shared DTOs and types between a Nest backend and an Angular or React frontend.
- **Microservices** — Nest has built-in transports for gRPC, Kafka, RabbitMQ and Redis, not only HTTP.

---

## 8. Input, Processing, Output

**Input:** an HTTP request (or a message from a queue transport).
**Processing:** middleware → guards → interceptors → pipes → controller → service → repository.
**Output:** a response, with exceptions mapped to status codes by exception filters.

---

## 9. Communication and Dependencies

- **Express or Fastify** — the underlying HTTP server
- **TypeScript with decorators** — `experimentalDecorators` and metadata are required
- **class-validator / class-transformer** — the standard validation pair
- **TypeORM, Prisma or Mongoose** — data access
- **Node.js** — same [event loop](../05%20-%20Frontend%20Architecture/Event%20Loop.md) constraints as any Node app

---

## 10. Alternatives

```text
NestJS       full structure, DI, decorators — Angular-like
    ↓
Express      no structure; you build the conventions
    ↓
Fastify      minimal and fast, with a good plugin system
    ↓
AdonisJS     batteries-included, more Laravel-like
    ↓
Spring Boot / ASP.NET Core   the same philosophy in Java and C#
```

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Use NestJS for large or long-lived TypeScript backends, teams of several developers, or projects where a shared structure matters more than minimalism.

> [!CAUTION]
> - **A small API with three endpoints** does not need modules, providers and decorators. The ceremony will outweigh the code.
> - **A team unfamiliar with dependency injection** faces a real learning curve — this is not "Express with types".
> - It is still Node underneath: **CPU-bound work blocks the event loop** exactly as it would anywhere else.

---

## 12. Advantages and Disadvantages

**Advantages**
- Consistent structure across every feature and every developer
- Real dependency injection, making testing straightforward
- TypeScript-first, with strong typing throughout
- Built-in support for microservice transports, GraphQL and WebSockets
- Excellent documentation and a coherent CLI

**Disadvantages**
- Substantial boilerplate for small projects
- Steep learning curve if DI and decorators are unfamiliar
- Heavy use of decorators and metadata reflection can obscure control flow
- More abstraction layers between a request and the database
- Slower startup than plain Express

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Throughput** | Slightly below raw Express; use the Fastify adapter to recover most of it |
| **Startup** | Slower — the DI container is built at boot |
| **Memory** | Higher than Express, still modest |
| **Concurrency** | Identical to Node — one loop per process |

---

## 14. Security Considerations

> [!CAUTION]
> **Guards are authorisation and must check the object, not only the role.** A guard confirming `user.role === 'user'` does not confirm that order 42 belongs to them. This is the most common serious API vulnerability, and framework structure does not prevent it.

- **`ValidationPipe` with `whitelist: true`** strips unexpected properties — the defence against mass assignment
- **Enable it globally**; a pipe applied per-controller will eventually be forgotten on one
- **Serialisation interceptors** (`@Exclude`) control what leaves — the equivalent of FastAPI's `response_model`
- **helmet, rate limiting and CORS** still need configuring; Nest does not add them for you
- Exception filters must not leak stack traces to clients

---

## 15. Mental Model

> [!NOTE]
> **NestJS is a building with a floor plan every architect must follow.**
>
> Express gives you an empty plot and total freedom. Nest gives you a plan: rooms of known purpose in known places. Constraining for a garden shed; essential when twenty builders work on the same tower and someone must be able to find the plumbing.

---

## 16. Mini Architecture Diagram

```text
Request
    ↓
Middleware
    ↓
Guard        → 403 if refused
    ↓
Interceptor  (before)
    ↓
Pipe         → 400/422 if validation fails
    ↓
Controller   (HTTP only)
    ↓
Service      (business logic)
    ↓
Repository → database
    ↓
Interceptor  (after: shape the response)
    ↓
Exception filter → status code
    ↓
Response
```

---

## 17. Complete Request Flow

```text
GET /users/42
    ↓
Global middleware (logging, helmet)
    ↓
AuthGuard verifies the JWT → attaches the user, or 401
    ↓
Interceptor starts a timing span
    ↓
ParseIntPipe converts "42" → 42, or 400
    ↓
Controller method called with typed arguments
    ↓
UsersService (injected) applies business rules
    ↓
Repository queries the database
    ↓
Serialisation interceptor strips @Exclude fields
    ↓
200 OK
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> NestJS trades boilerplate for a structure the framework enforces — worth it when many people maintain the codebase, and overhead when few do.

---

## 19. Common Mistakes

- **Using it for a tiny service** where the ceremony dominates
- **Business logic in controllers**, defeating the entire structure
- **`ValidationPipe` not applied globally**, so one endpoint is unvalidated
- **Guards that check roles but not object ownership**
- **Circular module dependencies**, which produce confusing runtime errors
- **Assuming it solves Node's blocking problem** — it does not
- **Skipping helmet and rate limiting** because the framework "feels" enterprise

---

## 20. Open Source Technologies

- **NestJS** — the framework and CLI
- **class-validator**, **class-transformer** — validation and serialisation
- **Prisma**, **TypeORM**, **Drizzle** — data access
- **@nestjs/swagger** — OpenAPI generation from decorators
- **@nestjs/microservices** — gRPC, Kafka, RabbitMQ transports

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Check that `ValidationPipe` is registered globally with `whitelist: true`.
- [ ] Find one guard that checks a role and verify whether it also checks object ownership.
- [ ] Take one controller and move any business logic out of it into a service.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Request → Guards → Pipes → Controller → Service → Repository → Database
```

## 2. Request Flow

```text
Input       an HTTP request or a queue message
    ↓
Processing  a fixed pipeline of guards, pipes, controller, service, repository
    ↓
Output      a serialised response, with exceptions mapped by filters
```

## 3. Real-World Usage

NestJS is most common in **companies with large TypeScript teams** who wanted the structural guarantees of Spring Boot without leaving the Node ecosystem. Its Angular-derived architecture is the point, not a coincidence.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | An opinionated, DI-based TypeScript backend framework |
| **Why does it exist?** | Because Express codebases lose their shape as teams grow |
| **Where does it belong?** | On top of Express or Fastify, structuring your application |
| **When should I use it?** | Large, long-lived TypeScript backends — not small services |
