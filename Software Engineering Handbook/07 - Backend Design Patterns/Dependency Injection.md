# Dependency Injection

> **In one line —** a class receives what it needs instead of creating it, which is what makes code testable without any mocking tricks.

| | |
|---|---|
| **Category** | Design Pattern |
| **Architectural Layer** | Application structure |
| **Abbreviation** | DI · IoC (Inversion of Control) |
| **Related notes** | [Services](Services.md) · [Repository Pattern](Repository%20Pattern.md) · [SOLID Principles](SOLID%20Principles.md) · [Clean Architecture](Clean%20Architecture.md) · [NestJS](../06%20-%20Backend%20Architecture/NestJS.md) |

---

## 1. Short Definition

*What is it?*

Dependency injection means an object does not construct its own collaborators. They are passed in — usually through the constructor — by whoever creates it.

---

## 2. Purpose

*What is its main purpose?*

To decouple a class from the concrete implementations it uses, so those implementations can be swapped: a real database in production, an in-memory one in tests, a cached one under load.

---

## 3. Problem

*What engineering problem does it solve?*

```text
WITHOUT DI                             WITH DI

class OrderService:                    class OrderService:
    def __init__(self):                    def __init__(self, repo: OrderRepo,
        self.repo = SqlOrderRepo()                       email: EmailSender):
        self.email = SmtpSender()              self.repo = repo
                                               self.email = email

↑ hard-wired to PostgreSQL and SMTP    ↑ works with anything satisfying
  testing it sends real email            the interface
  and needs a real database              tests pass fakes; no patching
```

> [!IMPORTANT]
> The test problem is the practical one. Without DI, testing means monkey-patching, `unittest.mock.patch` on import paths, or spinning up real infrastructure. With DI, a test constructs the object with fakes and runs in milliseconds.

---

## 4. Architecture Position

```text
Composition root  (application startup)
    ↓  creates and wires everything
Controller ← Service ← Repository ← database connection
    ↓
Each layer receives its dependencies; none constructs them
```

> [!TIP]
> **The composition root is the one place that knows the concrete types.** Everything else works with interfaces. Keeping that knowledge in one file is most of the benefit.

---

## 5. The three forms

```text
CONSTRUCTOR INJECTION    dependencies passed to the constructor
                         ← the default; makes requirements explicit and immutable

METHOD INJECTION         passed to a specific method
                         for something needed by one operation only

PROPERTY INJECTION       set after construction
                         avoid — the object can exist in an invalid state
```

---

## 6. DI containers

A container reads type declarations and wires the graph automatically.

```typescript
@Injectable()
export class OrderService {
  constructor(
    private readonly orders: OrderRepository,   // resolved by type
    private readonly email: EmailSender,
  ) {}
}
```

| Framework | Container |
|---|---|
| **Spring** | The original, annotation-driven |
| **ASP.NET Core** | Built in — `IServiceCollection` |
| **NestJS** | Built in, Angular-style |
| **FastAPI** | `Depends()` — functions, not classes |
| **Python / Go generally** | Often none; plain constructor arguments |

> [!TIP]
> **You do not need a container to do dependency injection.** Passing arguments to a constructor *is* DI. Containers help when the object graph is large; in a small application they add indirection for no gain.

---

## 7. Scopes

```text
SINGLETON     one instance for the application lifetime    stateless services, config
SCOPED        one per request                              database session, current user
TRANSIENT     a new one every time                         lightweight, stateful helpers
```

> [!CAUTION]
> **Injecting a scoped dependency into a singleton is a classic and dangerous bug.** The singleton captures the first request's database session and reuses it forever — producing data leaking between users, and errors that only appear under concurrency.

---

## 8. Real World Example

- **Spring** made DI mainstream in enterprise Java; the entire framework is a container.
- **ASP.NET Core** builds it into the framework, with no third-party library needed.
- **FastAPI's `Depends`** does the same thing with plain functions, and overriding a dependency in tests is a single line.
- **Go** deliberately has no container — the community passes structs to constructors and considers that sufficient.

---

## 9. Communication and Dependencies

- **The composition root** at startup wires everything
- **Interfaces / protocols** are what classes depend on
- **The framework's container**, when one is used
- **Tests** supply fakes through the same mechanism

---

## 10. When To Use / When NOT To Use

> [!TIP]
> Inject anything that touches the outside world — databases, HTTP clients, email, clocks, random number generators, file systems. Those are exactly the things you want to replace in tests.

> [!CAUTION]
> Do not inject value objects, pure functions or simple data structures. `datetime` should be injected as a clock **if you need to test time-dependent behaviour**; a `Money` class should just be constructed. Injecting everything produces a configuration file where the program used to be.

---

## 11. Advantages and Disadvantages

**Advantages**
- Testable without patching or real infrastructure
- Implementations swappable — cache, mock, alternative backend
- Dependencies are explicit and visible in the constructor signature
- Enforces the dependency-inversion principle naturally

**Disadvantages**
- Indirection — finding what actually runs takes an extra step
- Containers add configuration and, sometimes, runtime magic
- Container errors surface at startup or, worse, at first use
- Easy to over-apply until every class needs six injected collaborators

> [!IMPORTANT]
> **A constructor with six dependencies is a design signal, not a DI problem.** The class is doing too much. DI made the problem visible rather than causing it.

---

## 12. Performance Impact

| Aspect | Impact |
|---|---|
| **Runtime** | Negligible once resolved |
| **Startup** | Containers build the object graph at boot — noticeable in Spring, trivial elsewhere |
| **Compile-time DI** | Micronaut and Dagger resolve at build time, removing both cost and reflection |

---

## 13. Security Considerations

> [!CAUTION]
> **Scope confusion is a security bug, not merely a correctness one.** A singleton holding a request-scoped user context will serve one user's data to another. It is intermittent, load-dependent, and very hard to reproduce — the worst combination.

- **Never inject secrets as plain strings** into many classes; inject a configuration object or a secrets client so the blast radius is small
- **Container configuration is code** — a misconfigured binding can substitute a permissive implementation for a strict one
- **Test doubles must never reach production** — a container profile that swaps in a no-op authoriser is a real risk; fail loudly if the wrong profile is active

---

## 14. Mental Model

> [!NOTE]
> **Dependency injection is a chef given ingredients rather than sent shopping.**
>
> A chef who buys their own ingredients can only cook where their usual shop exists. A chef handed ingredients can cook anywhere, and you can hand them substitutes to test a recipe — which is exactly what a fake repository is.

---

## 15. Mini Architecture Diagram

```text
        Composition root (startup)
                 ↓ creates
        ┌────────┴─────────┐
   SqlOrderRepo      SmtpEmailSender
        └────────┬─────────┘
                 ↓ injected into
            OrderService
                 ↓ injected into
           OrderController

   In tests:
   InMemoryOrderRepo + FakeEmailSender → OrderService → assertions
```

---

## 16. Complete Request Flow

```text
Application starts
    ↓
Container registers: OrderRepository → SqlOrderRepository (scoped)
                     EmailSender     → SmtpSender (singleton)
    ↓
Request arrives
    ↓
Container creates a scoped database session
    ↓
Resolves OrderService, injecting the scoped repo and singleton sender
    ↓
Resolves OrderController, injecting the service
    ↓
Handler runs; nothing in the chain constructed its own dependencies
    ↓
Request ends → scoped instances disposed, session closed
```

---

## 17. Key Takeaway

> [!IMPORTANT]
> Give objects their dependencies instead of letting them construct their own — that single change is what makes code testable, and a container is optional convenience on top.

---

## 18. Common Mistakes

- **Injecting a scoped dependency into a singleton** — data leaking between requests
- **Using a container where constructor arguments would do**
- **Injecting everything**, including value objects
- **Six-dependency constructors** treated as normal rather than as a warning
- **Property injection**, allowing half-constructed objects
- **Depending on concrete classes** rather than interfaces, losing the benefit
- **Test-only bindings** reachable in production configuration

---

## 19. Open Source Technologies

- **Spring**, **ASP.NET Core DI**, **NestJS** — built-in containers
- **FastAPI `Depends`** — function-based DI
- **Dagger**, **Micronaut** — compile-time DI, no reflection
- **Wire** (Go) — code-generated wiring
- **pytest fixtures** — dependency injection for tests

---

## 20. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 21. Workbook Exercise

- [ ] Find one class that constructs its own database or HTTP client and inject it instead.
- [ ] Write a test for that class using a fake, with no patching.
- [ ] Check your container for any singleton holding a request-scoped dependency.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Composition root
    ↓
Controller ← Service ← Repository ← connection
```

## 2. Request Flow

```text
Input       a type or interface a class needs
    ↓
Processing  the container (or caller) supplies a concrete implementation
    ↓
Output      an object whose collaborators can be swapped freely
```

## 3. Real-World Usage

**ASP.NET Core** ships a DI container in the framework itself rather than as a library. Making injection the default path — not an optional pattern — is why testable structure is the norm in .NET codebases rather than an aspiration.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Passing dependencies in rather than constructing them internally |
| **Why does it exist?** | So implementations can be swapped, above all in tests |
| **Where does it belong?** | Throughout the application, wired at one composition root |
| **When should I use it?** | For anything touching the outside world — not for value objects |
