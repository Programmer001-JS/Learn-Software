# Backend Fundamentals

> **In one line —** the backend is the part nobody sees: it holds the truth, enforces the rules, and is the only place a decision can actually be trusted.

| | |
|---|---|
| **Category** | Overview note *(hub for this section)* |
| **Architectural Layer** | Server |
| **Related notes** | [Web Servers](Web%20Servers.md) · [Backend Frameworks](Backend%20Frameworks.md) · [Runtime Explained](Runtime%20Explained.md) · [Database Fundamentals](../08%20-%20Databases%20and%20Data/Database%20Fundamentals.md) · [Authentication](../09%20-%20Security/Authentication.md) |

---

## 1. Short Definition

The backend is the server-side part of an application: it receives requests, applies business rules, reads and writes the database, and returns responses. It is where the data actually lives and where the rules are actually enforced.

---

## 2. Why it exists

Three things cannot be done on a client, ever:

- **Store the truth** — the client's copy is one of many, and it can be edited
- **Enforce rules** — anything checked only in a browser can be bypassed with a single HTTP request
- **Hold secrets** — API keys, database credentials and signing keys must never reach a client

> [!IMPORTANT]
> **Everything the client sends is hostile until proven otherwise.** Not because your users are attackers, but because the request may not have come from your UI at all. This single assumption drives most backend design.

---

## 3. Architecture Position

```text
Browser / mobile / another service
    ↓  HTTPS
Load balancer / reverse proxy      (Nginx)
    ↓
Web server / app server            (Uvicorn, Gunicorn, Kestrel)
    ↓
Framework                          (FastAPI, Express, Spring Boot)
    ↓
Your code: routing → validation → auth → business logic
    ↓
Cache (Redis)  ·  Database (PostgreSQL)  ·  Queue  ·  Object storage
```

---

## 4. The anatomy of a request

```text
1. Connection accepted (TCP + TLS)
    ↓
2. Routing            which handler?
    ↓
3. Middleware         logging, CORS, rate limiting, request ID
    ↓
4. Authentication     who is this?
    ↓
5. Validation         is the input well-formed?
    ↓
6. Authorisation      may THIS user do THIS to THIS object?
    ↓
7. Business logic     the part that is actually your product
    ↓
8. Data access        cache → database
    ↓
9. Serialisation      build the response
    ↓
10. Response
```

> [!CAUTION]
> Steps 4 and 6 are different. Authentication is identity; authorisation is permission. Checking that someone is logged in but not that the record belongs to them is the single most common serious backend vulnerability.

---

## 5. The core responsibilities

| Responsibility | Note |
|---|---|
| **Data persistence** | [Databases](../08%20-%20Databases%20and%20Data/Database%20Fundamentals.md), transactions, migrations |
| **Business logic** | The rules that make it your product |
| **Authentication and authorisation** | [Folder 09](../09%20-%20Security/Authentication.md) |
| **Validation** | Never trust the client |
| **Integration** | Payments, email, third-party APIs |
| **Background work** | [Queues and workers](../10%20-%20Distributed%20Systems/Background%20Workers.md) |
| **Observability** | Logs, metrics, traces |

---

## 6. Stateless by default

```text
STATEFUL                          STATELESS
session in server memory          session in Redis or a token
    ↓                                 ↓
user must return to server 2      any server can serve any request
scaling and restarts hurt         add or remove servers freely
```

> [!TIP]
> Keep application servers stateless and push state into a shared store. This one decision is what makes horizontal scaling, rolling deploys and crash recovery straightforward.

---

## 7. I/O-bound, not CPU-bound

Most backends spend the overwhelming majority of their time **waiting** — for the database, another service, or the disk.

```text
Typical request:  2 ms of your code   +   40 ms waiting on the database
```

> [!IMPORTANT]
> This is why async models and connection pooling matter far more than language speed for most applications, and why "we rewrote it in a faster language" so often produces disappointing results. The bottleneck was never the language.

---

## 8. Real World Example

- **Instagram** ran on Django and PostgreSQL to enormous scale, because their bottleneck was media delivery rather than application code.
- **Stack Overflow** served vast traffic from a handful of servers, because heavy caching removed most database work.
- **Every mobile app** has a backend; the app is a view, the backend is the product.

---

## 9. Mental Model

> [!NOTE]
> **The frontend is the shop window; the backend is the warehouse, the till and the security office.**
>
> Customers only ever see the window. But the stock, the prices, the rules about who may take what, and the record of what happened all live in the back — and no amount of rearranging the window changes any of them.

---

## 10. Key Takeaway

> [!IMPORTANT]
> The backend is the only place rules can be enforced and truth can be stored — every check the frontend performs is a convenience that must be repeated on the server.

---

## 11. Common Mistakes

- **Trusting client input** — including from your own frontend
- **Checking authentication but not object ownership**
- **Putting business logic in controllers**, making it untestable and unreusable
- **Stateful servers** that block scaling
- **No timeouts** on outbound calls, so one slow dependency hangs everything
- **Secrets in source control**
- **No structured logging or request IDs**, making incidents unresolvable

---

## 12. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 13. Workbook Exercise

- [ ] Trace one request through your own backend and name every one of the ten steps in section 4.
- [ ] Find one endpoint that checks authentication but not object ownership.
- [ ] Measure how much of one request's time is your code versus waiting on I/O.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Client
    ↓
Reverse proxy
    ↓
App server → Framework → Your code
    ↓
Cache → Database
```

## 2. Request Flow

```text
Input       an HTTP request from an untrusted client
    ↓
Processing  route → authenticate → validate → authorise → execute → persist
    ↓
Output      a response, and a durable change to the truth
```

## 3. Real-World Usage

**Instagram** scaled to hundreds of millions of users on a conventional Django and PostgreSQL backend. Their architecture stayed simple because they correctly identified that their hard problem was image delivery, not application logic.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | The server-side application that owns data and enforces rules |
| **Why does it exist?** | Because clients cannot be trusted with truth, rules or secrets |
| **Where does it belong?** | Between clients and the database |
| **When should I use it?** | Whenever data must persist, be shared, or be protected |
