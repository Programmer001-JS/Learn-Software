# API Design

> **In one line —** an API is a promise to strangers; the hard part is not building it, it is never being able to take it back.

| | |
|---|---|
| **Category** | Design Discipline |
| **Architectural Layer** | Application boundary |
| **Related notes** | [HTTP HTTPS](HTTP%20HTTPS.md) · [gRPC](gRPC.md) · [Documentation Driven Development](../01%20-%20Foundation/07%20-%20Documentation%20Driven%20Development.md) · [Validation](../07%20-%20Backend%20Design%20Patterns/Validation.md) · [Authorization](../09%20-%20Security/Authorization.md) |

---

## 1. Short Definition

*What is it?*

API design is deciding what a system exposes to its callers: which operations exist, what they accept, what they return, how they fail, and how they change over time.

---

## 2. Purpose

*What is its main purpose?*

To create a **stable contract** so that the caller and the implementation can evolve independently. A good API lets you rewrite the entire backend without anyone noticing.

---

## 3. Problem

*What engineering problem does it solve?*

Once a client depends on your API, every detail you exposed becomes something you cannot change without breaking them.

> [!IMPORTANT]
> This is why API design is an **[architectural decision](../01%20-%20Foundation/03%20-%20Engineering%20Decision%20Making.md)**: it is a one-way door. Internal code can be refactored freely; a public API cannot.

---

## 4. Architecture Position

```text
Clients  (browser, mobile, partners, other services)
    ↓
━━━━━━━━━ API CONTRACT ━━━━━━━━━     ← the promise; stable
    ↓
Controllers → Services → Repositories → Database    ← the implementation; free to change
```

The whole value of the boundary is that everything below it can be replaced without touching anything above it.

---

## 5. Design principles

### Model resources, not actions

```text
BAD                                 GOOD
POST /getUser?id=42                 GET    /users/42
POST /updateUserEmail               PATCH  /users/42
POST /deleteUser                    DELETE /users/42
```

### Use the correct status codes

Returning `200 OK` with `{"error": "not found"}` inside makes it impossible for clients, proxies and monitoring to behave correctly.

### Be consistent

Pick `snake_case` or `camelCase` and never mix. Pick one date format — ISO 8601 with a timezone — and use it everywhere. Consistency is worth more than any individual choice being optimal.

### Design the errors as carefully as the successes

```json
{
  "error": "validation_failed",
  "message": "Email address is not valid",
  "field": "email",
  "request_id": "req_8f3a"
}
```

> [!TIP]
> A machine-readable `error` code, a human-readable `message`, and a `request_id` that appears in your logs. That last field turns "it broke" into a support ticket you can actually resolve.

---

## 6. Pagination, filtering, sorting

Any collection that can grow **must** be paginated from day one. Retrofitting pagination is a breaking change.

```text
CURSOR-BASED     ?limit=50&cursor=eyJpZCI6MTIzfQ
                 stable under inserts, scales well          ← prefer this
OFFSET-BASED     ?limit=50&offset=100
                 simple, but slow on large tables and skips rows when data shifts
```

---

## 7. Versioning

```text
URL path        /v1/users          explicit, visible, easy to route     ← most common
Header          Accept: application/vnd.api.v1+json    cleaner, less discoverable
```

> [!CAUTION]
> **Additive changes are safe; removals and renames are not.** Adding a field breaks nobody. Removing one, renaming one, or changing a type breaks every client that relied on it. Design assuming you can never delete anything.

---

## 8. Idempotency

Networks retry. Users double-click. Mobile clients resend after a timeout.

```text
POST /payments                     → charged twice on retry  ✗
POST /payments
Idempotency-Key: abc-123           → second call returns the first result  ✓
```

> [!IMPORTANT]
> Any operation that costs money, sends a message or creates a resource needs an idempotency key. This is not an edge case; it is the normal behaviour of unreliable networks.

---

## 9. Real World Example

- **Stripe** is widely considered the reference for API design: consistent naming, excellent errors, idempotency keys, dated versions, and a strong backwards-compatibility guarantee.
- **GitHub's API** has run for years with a clear versioning and deprecation process.
- **Twitter/X's API changes** are the counter-example — abrupt breaking changes destroyed an entire ecosystem of third-party clients.

---

## 10. REST vs GraphQL vs gRPC

| | REST | GraphQL | [gRPC](gRPC.md) |
|---|---|---|---|
| **Best for** | Public APIs, CRUD | Varied clients, complex reads | Internal services |
| **Over/under-fetching** | Common | Solved by design | Not really an issue |
| **Caching** | Excellent (HTTP) | Hard | None |
| **Learning curve** | Low | Medium | Medium |
| **Contract** | OpenAPI, optional | Schema, enforced | `.proto`, enforced |
| **Main risk** | Endpoint sprawl | Expensive nested queries | Not browser-friendly |

---

## 11. When To Use

> [!TIP]
> **REST** as the default for public and browser-facing APIs. **GraphQL** when many different clients need different shapes of the same data. **gRPC** for high-volume internal service communication.

---

## 12. When NOT To Use

> [!CAUTION]
> - **Do not expose your database schema as your API.** They have different lifecycles and different audiences; coupling them means every schema migration is a breaking API change.
> - **Do not build a public API before you have a consumer.** You will design the wrong thing and then be unable to change it.
> - **Do not add GraphQL to solve a problem you do not have** — for a single frontend consuming a well-designed REST API, it is added complexity.

---

## 13. Security Considerations

> [!CAUTION]
> **Broken Object Level Authorization is the number one API vulnerability** in the OWASP API Top 10. It is simply this: the endpoint checks that you are logged in, but not that the object belongs to you. `GET /orders/12345` returning someone else's order is the most common serious API bug there is.

- **Authorise every object access**, not just the endpoint
- **Validate all input** at the boundary; never trust a client, including your own
- **Rate-limit** everything, and return `429` with `Retry-After`
- **Never put tokens in query strings** — they end up in logs and browser history
- **Do not leak internals in errors** — no stack traces, no SQL, no file paths
- **Avoid mass assignment** — binding a whole request body to a model lets a caller set `is_admin`
- **Return 404 rather than 403** where the existence of a resource is itself sensitive

---

## 14. Advantages of designing deliberately

- Clients can be built in parallel with the backend
- The implementation stays free to change
- Errors become actionable rather than mysterious
- Documentation writes itself from the contract
- Fewer support requests, because behaviour is predictable

---

## 15. Mental Model

> [!NOTE]
> **An API is a restaurant menu.**
>
> Customers order from the menu, not from the kitchen. You can replace the chef, change suppliers or rebuild the kitchen entirely — as long as the dishes on the menu still arrive as described. Removing a dish people rely on is the one thing you cannot do quietly.

---

## 16. Mini Architecture Diagram

```text
Mobile app   Web app   Partner integration
      └─────────┼─────────┘
                ↓
        ━━━ API CONTRACT ━━━
                ↓
        Validation → AuthZ → Controller
                ↓
        Service layer (business rules)
                ↓
        Repository → Database
```

---

## 17. Complete Request Flow

```text
POST /v1/orders
Authorization: Bearer ...
Idempotency-Key: abc-123
    ↓
Rate limit check                → 429 if exceeded
    ↓
Authentication                  → 401 if the token is invalid
    ↓
Schema validation               → 422 with per-field errors
    ↓
Idempotency check               → return the stored result if this key was seen
    ↓
Authorisation on the objects    → 403 if the user may not act on them
    ↓
Business logic → database
    ↓
201 Created
Location: /v1/orders/42
{"id": 42, "status": "pending"}
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> An API is a promise you cannot withdraw — design the contract, the errors and the versioning strategy before writing the implementation.

---

## 19. Common Mistakes

- **Returning 200 for errors**
- **Exposing database columns directly** as the API shape
- **No pagination** on collections that will grow
- **Inconsistent naming and date formats** across endpoints
- **Breaking changes without a version bump**
- **Checking authentication but not object ownership** — the single most common serious API bug
- **Vague errors** — "something went wrong" is not a contract
- **No request ID**, making support and debugging guesswork

---

## 20. Open Source Technologies

- **OpenAPI / Swagger** — describe and document REST APIs formally
- **Postman**, **Bruno**, **httpie** — build and test requests
- **Spectral** — lint your API definition for consistency
- **Pact** — contract testing between services
- **GraphQL**, **Apollo** — schema-driven query APIs
- **Zod**, **Pydantic**, **FluentValidation** — validate at the boundary

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Take one endpoint of yours and check whether it authorises the *object*, not just the user.
- [ ] Write the error response format for your API and make every endpoint conform to it.
- [ ] Pick one collection endpoint without pagination and design how you would add it without breaking clients.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Clients
    ↓
API contract  (stable)
    ↓
Application  (free to change)
    ↓
Database
```

## 2. Request Flow

```text
Input       a request against a documented contract
    ↓
Processing  rate limit → authenticate → validate → authorise → execute
    ↓
Output      a predictable response, or a structured, actionable error
```

## 3. Real-World Usage

**Stripe** has maintained backwards compatibility for over a decade by pinning each account to the API version it integrated against. Old clients keep working untouched — an operational cost they accepted deliberately, and the reason their API is the industry reference.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | The designed contract between a system and its callers |
| **Why does it exist?** | So implementation and clients can evolve independently |
| **Where does it belong?** | At the boundary between your system and everyone else |
| **When should I use it?** | Whenever anything outside your codebase will call your code |
