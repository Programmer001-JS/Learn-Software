# Modular Monolith

> **In one line —** one deployment, hard internal boundaries: the discipline of microservices without the network, and the shape most systems should actually have.

| | |
|---|---|
| **Category** | Architecture Pattern |
| **Architectural Layer** | System |
| **Related notes** | [Monolith](Monolith.md) · [Microservices](Microservices.md) · [Domain Driven Design](../07%20-%20Backend%20Design%20Patterns/Domain%20Driven%20Design.md) · [Clean Architecture](../07%20-%20Backend%20Design%20Patterns/Clean%20Architecture.md) |

---

## 1. Short Definition

*What is it?*

A modular monolith is a single deployable application organised into **strictly bounded modules** that communicate only through explicit interfaces — never by reaching into each other's internals or database tables.

---

## 2. Purpose

*What is its main purpose?*

To get the maintainability benefits usually attributed to microservices — clear ownership, independent reasoning, replaceable components — while keeping transactions, function calls and a single deployment.

---

## 3. Problem

*What engineering problem does it solve?*

```text
UNSTRUCTURED MONOLITH                 MICROSERVICES
everything imports everything         boundaries enforced by the NETWORK
one change breaks three features          ↓
nobody knows what depends on what     but you pay: latency, sagas,
    ↓                                   tracing, orchestration, ops
becomes a big ball of mud

MODULAR MONOLITH
boundaries enforced by TOOLING, not by network calls
    ↓
clear ownership, no distributed systems tax
```

---

## 4. Architecture Position

```text
┌──────────── ONE DEPLOYMENT ────────────┐
│                                         │
│  ┌── Orders ──┐  ┌── Billing ─┐        │
│  │ public API │  │ public API │        │
│  │ internal   │  │ internal   │        │
│  │ own tables │  │ own tables │        │
│  └─────┬──────┘  └─────┬──────┘        │
│        └── explicit interfaces ──┘      │
│                                         │
└─────────────────┬───────────────────────┘
                  ↓
            One database
      (with per-module schemas or table ownership)
```

---

## 5. The rules that make it work

```text
1. Each module has a PUBLIC INTERFACE — a small, explicit surface
2. Nothing outside a module may import its internals
3. Each module OWNS its tables; others never query them directly
4. Cross-module communication is via interfaces or in-process events
5. The rules are ENFORCED BY TOOLING, not by agreement
```

> [!IMPORTANT]
> **Rule 5 is the whole difference.** A folder structure with a convention that "modules should not import each other's internals" decays within months, because nothing stops it at 6pm on a Friday. A lint rule that fails the build does stop it. Without enforcement, you have a regular monolith with optimistic folder names.

---

## 6. Enforcing boundaries

| Language | Tool |
|---|---|
| **Python** | `import-linter` — declare forbidden import paths |
| **Java** | **ArchUnit** — architecture rules as unit tests |
| **.NET** | **NetArchTest**, or separate projects with restricted references |
| **TypeScript** | ESLint `no-restricted-imports`, **dependency-cruiser** |
| **Ruby** | **Packwerk** — built by Shopify for exactly this |
| **Any** | Separate build modules with explicit dependency declarations |

```text
# import-linter contract
[importlinter:contract:modules]
type = independence
modules = orders, billing, shipping, identity
```

That configuration makes a forbidden import a **failed build**.

---

## 7. Database boundaries

```text
✗ Any module queries any table
✓ Each module owns its tables; others go through the owning module's interface
✓✓ A schema per module, with database permissions enforcing it
```

> [!TIP]
> Table ownership is the boundary people cheat on first, because a direct join is so convenient. It is also the boundary that makes extraction possible later — a module whose data nobody else touches can be lifted out; one whose tables are joined from four places cannot.

---

## 8. Real World Example

- **Shopify's Packwerk** was built specifically to modularise a very large Rails monolith rather than split it. It is the most public, best-documented example of this pattern at scale.
- **Teams that migrated back from microservices** typically land here: they kept the boundaries and dropped the network.
- **Spring Modulith** exists in the Java ecosystem for the same purpose, with verification and documentation generation.

---

## 9. The extraction path

This is the strategic argument for the pattern.

```text
Modular monolith
    ↓  a module genuinely needs independent deployment or scaling
Its public interface already exists
Its data is already isolated
Its dependencies are already explicit
    ↓
Extraction is a contained project
    ↓
─────────── versus ───────────
Big ball of mud → extraction means untangling first,
                  which is where most "we'll split it later" plans die
```

> [!IMPORTANT]
> **A modular monolith is a monolith that can become microservices. A big ball of mud is one that cannot.** That optionality is worth more than either endpoint, because you rarely know in advance which modules will need to move.

---

## 10. What you keep, and what you give up

```text
KEEP FROM THE MONOLITH              KEEP FROM MICROSERVICES
ACID transactions                   clear ownership boundaries
function-call latency               explicit interfaces
one stack trace                     independent reasoning
one deployment                      replaceable components
simple local development            enforced dependency direction

GIVE UP
independent deployment per team     ← the one genuine microservice benefit
independent scaling per module
per-module technology choice
```

---

## 11. When To Use / When NOT To Use

> [!TIP]
> **This should be the default for any non-trivial system.** It costs a lint rule and some discipline, and it preserves every option. For a new product with an unclear domain, it is strictly better than committing to service boundaries you cannot yet know are correct.

> [!CAUTION]
> It does not solve the organisational problem. If multiple teams are genuinely blocked by a shared deployment pipeline, no amount of internal modularity helps — that is when [microservices](Microservices.md) earn their cost.
>
> It also requires sustained discipline. Boundaries without enforcement erode, and a modular monolith that stopped being modular is just a monolith with more folders.

---

## 12. Advantages and Disadvantages

**Advantages**
- Clear ownership without distributed systems complexity
- Transactions still work across the application
- Refactoring across modules is one commit
- Cheap: one deployment, one runtime, one database
- Preserves the option to extract later
- Onboarding is easier — a new developer learns one module

**Disadvantages**
- Boundaries require tooling and discipline to hold
- No independent deployment
- Coarse scaling granularity
- One technology stack
- A shared failure domain
- Enforcement tooling is another thing to configure and maintain

---

## 13. Performance Impact

Identical to a [monolith](Monolith.md) — module boundaries are compile-time and lint-time constructs, not runtime ones. Cross-module calls are ordinary function calls.

> [!TIP]
> This is the point most easily missed: you get architectural separation **at zero runtime cost**. Microservices pay milliseconds per boundary crossing; a modular monolith pays nanoseconds.

---

## 14. Security Considerations

> [!CAUTION]
> **Module boundaries are not security boundaries.** They are enforced by the build, not by the operating system. A compromised code path inside the process can call anything in it, regardless of what the lint rules say.

- For genuine isolation — a compliance boundary, untrusted plugin code, or a component with a very different risk profile — you need a **separate process or service**
- **Per-module database schemas with distinct users** do add real defence in depth
- The advantage remains: **one deployment to patch, one dependency tree to scan** — microservices multiply both

---

## 15. Mental Model

> [!NOTE]
> **A modular monolith is one house with proper internal walls.**
>
> Not one open-plan room where every noise carries, and not separate buildings with the cost of separate plumbing. Rooms have doors and defined purposes, and you can add a real wall later if a room genuinely needs to become its own building.

---

## 16. Mini Architecture Diagram

```text
┌──────────────── ONE PROCESS ────────────────┐
│  Orders ──interface──► Billing              │
│    ↑                      │                 │
│    └──in-process events───┘                 │
│                                             │
│  each module: public API · internals · tables│
└───────────────────┬──────────────────────────┘
                    ↓
     Database — schema per module
                    ↑
     import-linter / ArchUnit fails the build on violations
```

---

## 17. Complete Request Flow

```text
POST /orders
    ↓
HTTP layer → Orders module's PUBLIC interface
    ↓
Orders needs stock: calls Inventory's PUBLIC interface
    ↓  a function call — nanoseconds, not a network hop
Inventory reads ITS OWN tables and returns
    ↓
BEGIN TRANSACTION
    order saved (orders schema)
    stock decremented (inventory schema)
COMMIT                      ← still atomic; this is what you keep
    ↓
OrderPlaced published as an IN-PROCESS event
    ↓
Billing and Notifications react — no broker involved
    ↓
201 returned in ~40 ms
    ↓
Someone tries to import inventory.internal.models from orders
    ↓
CI FAILS — the boundary is real
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> A modular monolith gives you enforced boundaries at zero runtime cost, and keeps the option to extract a module later — which is exactly what an unstructured monolith cannot offer.

---

## 19. Common Mistakes

- **Conventions without enforcement** — boundaries decay within months
- **Shared tables** across modules, which blocks any future extraction
- **Modules split by technical layer** rather than business capability
- **A "shared" or "common" module** that everything depends on, becoming a new ball of mud
- **Treating module boundaries as security boundaries**
- **Modules too small**, producing ceremony without benefit
- **Skipping straight to microservices** without ever establishing the boundaries

---

## 20. Open Source Technologies

- **Packwerk** (Ruby), **Spring Modulith** (Java), **import-linter** (Python)
- **ArchUnit**, **NetArchTest** — architecture rules as tests
- **dependency-cruiser**, ESLint `no-restricted-imports` — TypeScript
- **Nx**, **Turborepo** — enforced module boundaries in monorepos

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Draw your current modules and mark every import that crosses a boundary it should not.
- [ ] Add one enforcement rule to CI and see how many violations already exist.
- [ ] Pick one module and ask: could it be extracted today? What would block it?

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
One deployment → modules with public interfaces → schema per module → one database
```

## 2. Request Flow

```text
Input       a request routed to one module's public interface
    ↓
Processing  in-process calls across enforced boundaries, in one transaction
    ↓
Output      a fast response — with architecture violations failing the build
```

## 3. Real-World Usage

**Shopify built Packwerk** to modularise a very large Rails monolith instead of splitting it into services. Choosing enforcement tooling over distribution, at that scale, is the strongest available endorsement of this pattern.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | One deployment with strictly enforced internal module boundaries |
| **Why does it exist?** | To get microservice discipline without the distributed systems cost |
| **Where does it belong?** | As the default architecture for non-trivial systems |
| **When should I use it?** | Almost always — moving on only when teams need independent deployment |
