# Routing

> **In one line —** matching an incoming URL and method to the function that handles it, and extracting the parameters along the way.

| | |
|---|---|
| **Category** | Architectural Pattern |
| **Architectural Layer** | Application, request pipeline |
| **Related notes** | [Middleware](Middleware.md) · [Controllers](Controllers.md) · [API Design](../04%20-%20Networking%20and%20Internet/API%20Design.md) · [HTTP HTTPS](../04%20-%20Networking%20and%20Internet/HTTP%20HTTPS.md) |

---

## 1. Short Definition

*What is it?*

Routing is the mechanism that decides which handler runs for a given HTTP method and path, and which parts of the URL become parameters.

```text
GET /users/42/orders?status=pending
     ↓
handler: getUserOrders
params: user_id = 42
query:  status = "pending"
```

---

## 2. Purpose

*What is its main purpose?*

To turn a text URL into a function call with typed arguments, so handlers never parse URLs themselves.

---

## 3. Problem

*What engineering problem does it solve?*

Without routing, every handler would inspect the raw URL string. Matching, ordering, parameter extraction and 404 handling would be reimplemented — badly — in every project.

---

## 4. Architecture Position

```text
Request
    ↓
Middleware
    ↓
ROUTER          ← you are here: match method + path
    ↓
Controller / handler
    ↓
Service → repository
```

---

## 5. The parts of a route

```text
POST /api/v1/users/42/orders?limit=10
 │      │       │    │    │        │
 │      │       │    │    │        └── query string: filters, pagination
 │      │       │    │    └── nested resource
 │      │       │    └── path parameter
 │      │       └── resource
 │      └── version prefix
 └── method: determines the intent
```

| Part | Use for |
|---|---|
| **Method** | The action — GET, POST, PUT, PATCH, DELETE |
| **Path** | Identifying the resource |
| **Path parameter** | A required identifier |
| **Query string** | Optional filtering, sorting, pagination |
| **Body** | Data being sent |

> [!TIP]
> **A path identifies a thing; a query string modifies how you see it.** `/users/42` is a user; `?fields=name` changes the view of it. Anything required belongs in the path.

---

## 6. Route matching order

Most routers match in **registration order**, first match wins.

```text
BROKEN                            CORRECT
GET /users/:id                    GET /users/me
GET /users/me     ← unreachable   GET /users/:id
                                     ↑ specific before generic
```

> [!CAUTION]
> `/users/me` registered after `/users/:id` never runs — `:id` matches `"me"` first, and your handler tries to look up a user with the id `"me"`. This is one of the most common routing bugs there is.

---

## 7. RESTful conventions

```text
GET    /orders          list
POST   /orders          create
GET    /orders/42       read one
PUT    /orders/42       replace
PATCH  /orders/42       modify
DELETE /orders/42       delete
GET    /orders/42/items nested collection
```

> [!TIP]
> Nest at most one level. `/users/1/orders/2/items/3/comments` is unreadable and couples the URL to a hierarchy that will change. Prefer `/items/3/comments`.

---

## 8. Real World Example

- **File-based routing** — [Next.js](../05%20-%20Frontend%20Architecture/Next.js.md) derives routes from the folder structure, so the URL map is visible in the file tree.
- **Decorator routing** — Spring, NestJS and FastAPI attach routes to methods, keeping the URL beside the code.
- **API gateways** route by path prefix to entirely different services.
- **Versioning** (`/v1/`, `/v2/`) is routing used to keep an old contract alive while a new one exists.

---

## 9. Input, Processing, Output

**Input:** an HTTP method and a path.
**Processing:** matched against registered patterns, in order; parameters extracted and often type-converted.
**Output:** a handler invocation with typed arguments — or a `404` if nothing matched, or a `405` if the path matched but the method did not.

---

## 10. Communication and Dependencies

- **[Middleware](Middleware.md)** runs before and around it
- **[Controllers](Controllers.md)** are what it dispatches to
- **The framework** — every one has its own syntax and precedence rules

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Keep routes thin: match, extract, dispatch. All logic belongs in the controller and below.

> [!CAUTION]
> Do not encode behaviour in the URL — `/users/42/activate` and `/users/42/deactivate` are RPC dressed as REST. Prefer `PATCH /users/42 {"active": true}`. And do not put anything sensitive in a path or query string; **URLs are logged everywhere**, by your server, every proxy in between, and the browser's history.

---

## 12. Advantages and Disadvantages

**Advantages**
- One clear map from URL to code
- Parameter extraction and conversion handled for you
- Enables versioning, grouping and per-route middleware
- Predictable URLs make an API learnable

**Disadvantages**
- Ordering bugs are silent
- Deeply nested routes become unreadable
- Route explosion in large applications without grouping
- Different frameworks disagree about precedence rules

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Matching** | Negligible — modern routers use tries or compiled regexes |
| **Route count** | Thousands of routes are fine |
| **Wildcards** | Overly broad patterns can shadow specific routes |

Routing is essentially never a performance problem. It is a **correctness and clarity** concern.

---

## 14. Security Considerations

> [!CAUTION]
> **A route existing is not permission to call it.** Routing decides *which* handler runs, never *whether the caller may*. Every route needs an authorisation check that verifies the specific object, not only the user's role.

- **Never put tokens or secrets in URLs** — they appear in access logs, proxy logs and browser history
- **Path traversal** — a route parameter used to build a file path must be validated
- **Enumeration** — sequential IDs in URLs let attackers walk your data; use UUIDs where that matters
- **Wildcard routes** can accidentally expose internal endpoints; review them
- **404 vs 403** — returning 403 for a resource that exists reveals its existence; return 404 when that matters

---

## 15. Mental Model

> [!NOTE]
> **Routing is a building directory in a lobby.**
>
> The board tells you that Accounts is on the third floor. It does not check whether you are allowed in — that is the door at the top of the stairs. Confusing the directory with the lock is exactly the mistake described above.

---

## 16. Mini Architecture Diagram

```text
GET /api/v1/users/42/orders?status=pending
    ↓
Middleware
    ↓
┌───────── ROUTER ─────────┐
│  method match: GET       │
│  path match: /users/:id/orders │
│  extract: id = 42        │
│  extract: status = pending│
└────────────┬─────────────┘
             ↓
    getUserOrders(id=42, status="pending")
             ↓
      Controller → Service → Repository
```

---

## 17. Complete Request Flow

```text
Request arrives
    ↓
Middleware chain runs
    ↓
Router walks registered routes in order
    ↓
No path match       → 404
Path but not method → 405
    ↓
Match found: parameters extracted and converted ("42" → 42)
    ↓
Conversion fails → 400 before the handler ever runs
    ↓
Route-specific middleware (if any)
    ↓
Controller invoked with typed arguments
    ↓
AUTHORISATION checked here — routing did not do it
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> Routing maps a URL to a handler and extracts parameters — it never decides whether the caller is allowed, and specific routes must be registered before generic ones.

---

## 19. Common Mistakes

- **Generic routes registered before specific ones** — `/users/:id` shadowing `/users/me`
- **Assuming a route implies authorisation**
- **Verbs in URLs** — `/getUser`, `/users/42/activate`
- **Deep nesting** that couples URLs to a changing hierarchy
- **Tokens in query strings**, ending up in logs
- **Unvalidated path parameters** used in file paths or queries
- **Inconsistent conventions** — singular here, plural there

---

## 20. Open Source Technologies

- **Express Router**, **Fastify**, **Hono** — JavaScript
- **FastAPI**, **Django URLs**, **Flask** — Python
- **Spring `@RequestMapping`**, **ASP.NET Core routing** — JVM and .NET
- **Next.js file-based routing**
- **OpenAPI** — documents the route map formally

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] List your routes in registration order and look for any generic route shadowing a specific one.
- [ ] Check that every route has an authorisation check that verifies object ownership.
- [ ] Find any endpoint accepting a token in the query string and move it to a header.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Request → Middleware → Router → Controller → Service → Repository
```

## 2. Request Flow

```text
Input       an HTTP method and path
    ↓
Processing  matched in order, parameters extracted and converted
    ↓
Output      a handler call with typed arguments, or 404/405
```

## 3. Real-World Usage

**Next.js file-based routing** makes the route map the folder structure. It removes an entire class of bugs — a route cannot be registered in the wrong order because there is no registration order.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Matching method and path to a handler, with parameter extraction |
| **Why does it exist?** | So handlers never parse URLs themselves |
| **Where does it belong?** | Between middleware and controllers |
| **When should I use it?** | Every HTTP application — keeping routes thin and authorisation separate |
