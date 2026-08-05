# Express.js

> **In one line —** a thin layer of routing and middleware over Node's HTTP server; it decides almost nothing for you, which is both why it won and why large Express codebases diverge.

| | |
|---|---|
| **Category** | Web Framework |
| **Architectural Layer** | Application |
| **Language** | JavaScript / TypeScript (Node.js) |
| **Related notes** | [Backend Frameworks](Backend%20Frameworks.md) · [NestJS](NestJS.md) · [Node.js Runtime](../03%20-%20Programming%20Languages%20and%20Runtime/Node.js%20Runtime.md) · [Middleware](../07%20-%20Backend%20Design%20Patterns/Middleware.md) |

---

## 1. Short Definition

*What is it?*

Express is a minimal web framework for [Node.js](../03%20-%20Programming%20Languages%20and%20Runtime/Node.js%20Runtime.md). It provides routing, a middleware pipeline, and small conveniences over the built-in HTTP module — and nothing else.

---

## 2. Purpose

*What is its main purpose?*

To make Node's raw HTTP server usable without imposing any structure. You bring the ORM, the validation library, the auth strategy and the project layout.

---

## 3. Problem

*What engineering problem does it solve?*

Node's built-in server gives you a request and a response object and leaves URL matching, body parsing and error handling entirely to you.

```text
RAW NODE                              EXPRESS
if (req.url === '/users'              app.get('/users/:id', handler)
    && req.method === 'GET') ...
manual parsing, manual routing        routing, params, middleware
```

---

## 4. Architecture Position

```text
Nginx
    ↓
Node.js process (one event loop per process)
    ↓
EXPRESS         ← routing + middleware chain
    ↓
Your handlers
    ↓
Database
```

---

## 5. The middleware model — the whole design

```javascript
app.use(express.json());               // parse bodies
app.use(cors());                       // CORS headers
app.use(authenticate);                 // custom auth
app.get('/users/:id', getUser);        // route handler
app.use(errorHandler);                 // error middleware, last
```

```text
Request
    ↓ → middleware 1 → next()
    ↓ → middleware 2 → next()
    ↓ → route handler → res.send()
    ↓ → error middleware (only if next(err) was called)
Response
```

> [!IMPORTANT]
> Everything in Express is middleware, including routes. Order matters absolutely — a middleware registered after a route that sends a response never runs. Most "why isn't my auth applying" bugs are ordering bugs.

---

## 6. Real World Example

- **The default choice** for Node APIs for over a decade; an enormous number of production services run on it.
- **Its middleware ecosystem** (helmet, morgan, passport, multer) is the real product — the framework is small, the ecosystem is vast.
- **Many "Node backends"** are actually Express plus a dozen npm packages assembled by convention.

---

## 7. Input, Processing, Output

**Input:** an HTTP request, arriving on the Node event loop.
**Processing:** matched against routes; passed through the middleware chain in registration order.
**Output:** whatever a handler writes to the response object.

---

## 8. Async error handling — the classic trap

```javascript
// Express 4: an async rejection is NOT caught — the request hangs forever
app.get('/x', async (req, res) => {
  const data = await mayThrow();       // ✗ unhandled rejection
  res.json(data);
});

// Fix: wrap it, or use express-async-errors, or Express 5
app.get('/x', async (req, res, next) => {
  try { res.json(await mayThrow()); }
  catch (err) { next(err); }
});
```

> [!CAUTION]
> In Express 4 an unhandled promise rejection in a handler does not reach your error middleware. The request simply never responds, and the client eventually times out. Express 5 fixes this; until you are on it, wrap every async handler.

---

## 9. Communication and Dependencies

- **Node.js** — the runtime and its [event loop](../05%20-%20Frontend%20Architecture/Event%20Loop.md)
- **npm middleware** — for essentially every feature
- **An ORM or driver** — Prisma, Sequelize, Knex, `pg`
- **A process manager** — PM2 or an orchestrator, one process per core

---

## 10. Alternatives

```text
Express      the default; minimal, huge ecosystem, aging design
    ↓
Fastify      faster, schema-based validation, modern plugin system
    ↓
Koa          from the Express authors; async-first, even more minimal
    ↓
Hono         very fast, edge-runtime friendly
    ↓
NestJS       structure and opinions on top (often Express underneath)
```

> [!TIP]
> For a **new** Node API, **Fastify** is usually the better default: built-in JSON schema validation, better async handling, and measurably faster. Express's advantage is familiarity and the sheer size of its ecosystem.

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Use Express for small to medium APIs, for prototypes, and when the team already knows it. Its minimalism is genuinely valuable when you want full control.

> [!CAUTION]
> On a large, long-lived codebase, its lack of opinions becomes a liability: every developer structures things differently, and there is no framework-enforced pattern to fall back on. That is exactly the gap [NestJS](NestJS.md) exists to fill.

---

## 12. Advantages and Disadvantages

**Advantages**
- Tiny, easy to learn in an afternoon
- Enormous middleware ecosystem
- Complete freedom over structure and libraries
- Battle-tested over more than a decade

**Disadvantages**
- No structure — large codebases drift
- No built-in validation, DI, or ORM
- Async error handling is broken in v4
- Development has been slow; many defaults are dated
- Every project reinvents its own conventions

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Throughput** | Good, but noticeably behind Fastify |
| **Concurrency** | One event loop per process — use cluster for cores |
| **Memory** | Very light |
| **CPU** | Any blocking handler stalls all concurrent requests |

---

## 14. Security Considerations

> [!CAUTION]
> Express ships with **almost no security defaults**. Unlike Django or Laravel, nothing is protected until you add it — no CSRF, no security headers, no rate limiting, no body size limits.

- **`helmet`** for security headers — add it on day one
- **`express-rate-limit`** for brute-force protection
- **Body size limits**: `express.json({ limit: '100kb' })`
- **Validate everything** — Zod, Joi or a schema library; Express does nothing here
- **`app.disable('x-powered-by')`** to reduce fingerprinting
- **Audit your middleware** — every npm package runs with full process privileges

---

## 15. Mental Model

> [!NOTE]
> **Express is an assembly line where you install every station yourself.**
>
> The framework provides the conveyor belt and the rule that each station passes the item along. Which stations exist, what order they stand in, and whether anyone checks the product for defects is entirely up to you.

---

## 16. Mini Architecture Diagram

```text
Request
    ↓
helmet → cors → json parser → logger → rate limiter
    ↓
authenticate → authorise
    ↓
route handler
    ↓
service → repository → database
    ↓
error middleware (catches next(err))
    ↓
Response
```

---

## 17. Complete Request Flow

```text
GET /users/42 arrives on the event loop
    ↓
Middleware chain runs in registration order
    ↓
Auth middleware verifies the token → next() or next(err)
    ↓
Route matched; req.params.id populated
    ↓
Handler: await db.getUser(42)  → loop serves other requests meanwhile
    ↓
res.json(user)
    ↓
Any next(err) along the way → error middleware → status code
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> Express gives you routing and an ordered middleware chain and nothing else — which means every security control and every structural convention is something you must add deliberately.

---

## 19. Common Mistakes

- **Unwrapped async handlers** in Express 4 — requests that hang forever
- **Middleware in the wrong order**, so auth never applies to a route
- **No `helmet`, no rate limiting, no body size limit**
- **Business logic inside route handlers**
- **Forgetting `next()`**, leaving the request hanging
- **One process on a multi-core machine**
- **Adding npm packages casually** without weighing supply-chain risk

---

## 20. Open Source Technologies

- **Express**, **Fastify**, **Koa**, **Hono** — frameworks
- **helmet**, **cors**, **morgan**, **express-rate-limit** — essential middleware
- **Zod**, **Joi** — validation
- **Prisma**, **Drizzle**, **Knex** — data access
- **PM2** — clustering and process management

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Check whether every async handler in your app is wrapped or running on Express 5.
- [ ] Add helmet, a rate limiter and a body size limit, and verify each works.
- [ ] Draw your middleware chain in order and identify anything registered too late.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Nginx → Node process → Express middleware chain → handlers → database
```

## 2. Request Flow

```text
Input       an HTTP request
    ↓
Processing  passed through middleware in registration order to a route handler
    ↓
Output      a response, or an error routed to error middleware
```

## 3. Real-World Usage

Express became the default Node framework by deciding almost nothing. That minimalism made it universal — and is also why **NestJS** and **Fastify** both exist, each adding back a different thing Express left out.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A minimal routing and middleware framework for Node.js |
| **Why does it exist?** | Because raw Node HTTP handling is too low-level to build on |
| **Where does it belong?** | Between the Node runtime and your handlers |
| **When should I use it?** | Small to medium APIs — consider Fastify for new work, NestJS for large teams |
