# Authorization

> **In one line —** deciding what an authenticated identity may do — and forgetting to check *which object* is the most common serious API vulnerability in existence.

| | |
|---|---|
| **Category** | Security Concept |
| **Architectural Layer** | Application |
| **Abbreviation** | AuthZ |
| **Related notes** | [Authentication](Authentication.md) · [Services](../07%20-%20Backend%20Design%20Patterns/Services.md) · [OWASP Top 10](OWASP%20Top%2010.md) · [API Design](../04%20-%20Networking%20and%20Internet/API%20Design.md) |

---

## 1. Short Definition

*What is it?*

Authorisation decides whether a known identity is permitted to perform a specific action on a specific resource.

```text
Authentication:  you are user 42                 → 401 if unknown
Authorization:   may user 42 delete order 99?    → 403 if not allowed
```

---

## 2. Purpose

*What is its main purpose?*

To enforce the rules about who may see and change what — the boundary between a multi-user system that works and one that leaks.

---

## 3. Problem

*What engineering problem does it solve?*

```text
GET /orders/12345
    ↓
Middleware: is the user logged in?  ✓ yes
    ↓
Handler returns order 12345
    ↓
...which belongs to a different customer entirely
```

> [!CAUTION]
> This is **Broken Object Level Authorization**, and it is consistently ranked the **number one API vulnerability** by OWASP. The endpoint checks authentication, and nobody checks ownership. Changing a number in a URL is not a sophisticated attack — it is the most common one that works.

---

## 4. Architecture Position

```text
Authentication      identity established
    ↓
ROUTE-LEVEL AUTHZ   may this role reach this endpoint at all?
    ↓
Controller
    ↓
Service
    ↓
OBJECT-LEVEL AUTHZ  does THIS record belong to THIS user?      ← the critical one
    ↓
Repository → database
```

> [!IMPORTANT]
> **Both layers are required.** Route-level checks alone are what produce the vulnerability above. Object-level checks belong in the [service layer](../07%20-%20Backend%20Design%20Patterns/Services.md), not the controller — otherwise a background job or CLI command bypasses them entirely.

---

## 5. The models

```text
RBAC — Role-Based
    user has roles → roles have permissions
    "admins may delete orders"
    ✓ simple, understandable, covers most applications
    ✗ role explosion as exceptions accumulate

ABAC — Attribute-Based
    decisions from attributes of user, resource, and context
    "managers may approve expenses under €5,000 in their own department"
    ✓ expressive
    ✗ hard to reason about and to audit

ReBAC — Relationship-Based
    "may view IF owner OR member of the owning team"
    ✓ natural for sharing and collaboration
    ✗ needs a graph; see Google Zanzibar / OpenFGA

ACL — Access Control List
    explicit per-object permissions
    ✓ precise   ✗ does not scale to many objects
```

> [!TIP]
> Start with **RBAC plus an ownership check**. That combination covers the large majority of applications, and it is far easier to audit than a policy engine nobody fully understands.

---

## 6. Where checks must live

```text
✗ FRONTEND ONLY          hiding a button is UX, not security —
                         anyone can call the API directly

✗ CONTROLLER ONLY        background jobs and CLI commands bypass it

✓ SERVICE LAYER          every entry point goes through it

✓✓ DATABASE LAYER        row-level security or a repository that always
                         filters by tenant — cannot be forgotten
```

> [!IMPORTANT]
> **A hidden UI element is not an access control.** The most common way this fails: the button is hidden for non-admins, and the endpoint it called never checks anything.

---

## 7. Multi-tenancy — where this gets dangerous

```text
✗ Every query must remember:  WHERE tenant_id = :tenant
    ↓
One query written without it → cross-tenant data leak

✓ Enforce it once, structurally:
    PostgreSQL Row-Level Security
    a repository that injects the filter automatically
    EF Core global query filters
```

> [!CAUTION]
> Relying on developers to remember a `WHERE` clause on every query is not a control — it is a hope. In a multi-tenant system, enforce tenant isolation at a layer that cannot be bypassed.

---

## 8. Real World Example

- **BOLA / IDOR** — sequential IDs in URLs, no ownership check — is the archetypal API breach, reported constantly in bug bounty programmes.
- **Google Zanzibar** is the reference design for relationship-based authorisation at scale; **OpenFGA** and **SpiceDB** are open implementations.
- **PostgreSQL Row-Level Security** enforces tenancy in the database, where application bugs cannot bypass it.

---

## 9. Fail closed

```python
# ✗ fails OPEN — an unknown role gets through
if user.role == "banned":
    deny()
allow()

# ✓ fails CLOSED — anything not explicitly permitted is denied
if not policy.allows(user, action, resource):
    deny()
allow()
```

> [!IMPORTANT]
> **Deny by default.** New roles, new endpoints and unexpected states must be refused rather than permitted. Every allow-list is safe by construction; every deny-list eventually misses a case.

---

## 10. When To Use / When NOT To Use

> [!TIP]
> Check authorisation on **every** request that touches data belonging to someone. Put the check where every entry point passes through it, and prefer structural enforcement over remembering.

> [!CAUTION]
> Do not adopt a policy engine before you have outgrown RBAC. Externalised policy is powerful and adds a second system that must be understood, tested and kept in sync — and a policy nobody can read is not auditable.

---

## 11. Advantages and Disadvantages

**Advantages of explicit, centralised authorisation**
- One place to read, review and test the rules
- Consistent across HTTP, jobs and CLI
- Auditable — you can answer "who may do this?"
- New endpoints inherit the model rather than reinventing it

**Disadvantages**
- Per-object checks add queries
- Fine-grained models become hard to reason about
- Policy engines add operational complexity
- Rules scattered across layers are worse than none, because they create false confidence

---

## 12. Performance Impact

| Aspect | Impact |
|---|---|
| **Role check** | Free — the role is already on the identity |
| **Object ownership** | Usually already loading the record; check it then |
| **ReBAC graph traversal** | Real cost; caching required at scale |
| **Row-level security** | Adds a predicate to every query — usually negligible |

> [!TIP]
> Combine the load and the check: fetch the record filtered by owner (`WHERE id = ? AND user_id = ?`) rather than fetching it and comparing afterwards. Not found and not permitted then behave identically, which is also better for information disclosure.

---

## 13. Security Considerations

> [!CAUTION]
> Beyond the object-level problem, three failures recur:

- **Privilege escalation through mass assignment** — a request body containing `"role": "admin"` bound directly to a model. Bind to an explicit input schema
- **Horizontal escalation** — accessing a peer's data (the BOLA case)
- **Vertical escalation** — a normal user reaching an admin function because the check was only in the UI

Also:

- **404 versus 403** — returning 403 confirms a resource exists; return 404 when existence itself is sensitive
- **Re-check on every request** — permissions change, and a long-lived token may carry stale claims
- **Log authorisation denials** — repeated 403s from one account is an attack signal
- **Test negatively** — most test suites verify that permitted actions succeed and never that forbidden ones fail

---

## 14. Mental Model

> [!NOTE]
> **Authentication is the passport; authorisation is the visa and the ticket.**
>
> The passport proves who you are. The visa says which country you may enter, and the ticket says which seat is yours. Checking the passport and then letting anyone sit in any seat is precisely the object-level authorisation bug.

---

## 15. Mini Architecture Diagram

```text
Request + identity
    ↓
Route policy      "may this role call this endpoint?"      → 403
    ↓
Service
    ↓
Object policy     "is this record theirs?"                 → 403 or 404
    ↓
Repository — filters by tenant/owner structurally
    ↓
Database — row-level security as the last line
```

---

## 16. Complete Request Flow

```text
DELETE /orders/12345
    ↓
Authenticated: user 42
    ↓
Route policy: may the "customer" role delete orders?  → yes, in principle
    ↓
Service loads order 12345 FILTERED BY OWNER:
    SELECT ... WHERE id = 12345 AND user_id = 42
    ↓
No row → 404  (not "403", which would confirm the order exists)
    ↓
Row found → the ownership check is already satisfied by construction
    ↓
Additional rule: shipped orders cannot be deleted → 409
    ↓
Deleted; the authorisation decision is logged
```

---

## 17. Key Takeaway

> [!IMPORTANT]
> Authorisation must check the specific object, not only the role — and it belongs in the service or data layer, where no entry point can bypass it.

---

## 18. Common Mistakes

- **Checking authentication but not object ownership** — the number one API vulnerability
- **Hiding UI elements** and treating that as access control
- **Authorisation only in the controller**, bypassed by jobs and CLI
- **Fail-open logic** — allowing anything not explicitly denied
- **Mass assignment** letting a caller set their own role
- **Remembering the tenant filter** instead of enforcing it structurally
- **No negative tests** — nobody verifies that forbidden actions actually fail
- **403 where 404 would not confirm existence**

---

## 19. Open Source Technologies

- **Casbin**, **OpenFGA**, **SpiceDB**, **Oso** — authorisation engines
- **Open Policy Agent (OPA)** — policy as code
- **PostgreSQL Row-Level Security** — enforcement in the database
- **Django Guardian**, **Pundit**, **CASL**, **Spring Security `@PreAuthorize`**
- **OWASP API Security Top 10** — BOLA is item one

---

## 20. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 21. Workbook Exercise

- [ ] Take one endpoint that accepts a resource ID and try it with another user's ID. What happens?
- [ ] Check whether a background job in your system could perform an action the API would refuse.
- [ ] Write one negative test asserting that a forbidden action returns 403 or 404.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Identity → route policy → service → object policy → repository (tenant filter) → DB (RLS)
```

## 2. Request Flow

```text
Input       an authenticated identity plus a requested action and resource
    ↓
Processing  role check, then object-level ownership check, denying by default
    ↓
Output      the action performed, or 403/404 with the decision logged
```

## 3. Real-World Usage

**Broken Object Level Authorization** tops the OWASP API Security Top 10 because it is both trivial to exploit — change a number in a URL — and extremely common. It is the single highest-value thing to audit in any API.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Deciding what a known identity may do to a specific resource |
| **Why does it exist?** | Because knowing who someone is says nothing about what they may access |
| **Where does it belong?** | The service and data layers, not the UI or the controller alone |
| **When should I use it?** | On every request touching someone's data — failing closed by default |
