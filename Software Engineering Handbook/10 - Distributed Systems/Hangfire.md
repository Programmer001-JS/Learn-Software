# Hangfire

> **In one line —** .NET background jobs stored in the database you already have: no broker, transactional enqueue, and a dashboard that comes free.

| | |
|---|---|
| **Category** | Background Job Framework |
| **Architectural Layer** | Application |
| **Language** | C# / .NET |
| **Storage** | SQL Server, PostgreSQL, Redis |
| **Related notes** | [Background Workers](Background%20Workers.md) · [ASP.NET Core](../06%20-%20Backend%20Architecture/ASP.NET%20Core.md) · [Transactions](../08%20-%20Databases%20and%20Data/Transactions.md) · [Message Queues](Message%20Queues.md) |

---

## 1. Short Definition

*What is it?*

Hangfire is a background job framework for .NET that stores jobs in a **database** rather than a message broker, and includes a built-in web dashboard.

---

## 2. Purpose

*What is its main purpose?*

To provide reliable background processing with no additional infrastructure — the jobs live in the same database as your application data.

---

## 3. Problem

*What engineering problem does it solve?*

```text
TRADITIONAL                           HANGFIRE
deploy RabbitMQ or Redis              use the existing database
configure a worker framework          one NuGet package
build a monitoring dashboard          dashboard included
    ↓                                     ↓
another system to operate             nothing new to run
```

> [!IMPORTANT]
> The **transactional enqueue** is the deeper advantage. Because the job is a database row, you can enqueue it inside the same [transaction](../08%20-%20Databases%20and%20Data/Transactions.md) as your business write. Either both happen or neither does — which removes the "the order saved but the email job vanished" class of bug entirely.

---

## 4. Architecture Position

```text
ASP.NET Core application
    ↓  BackgroundJob.Enqueue(...)
Database  (SQL Server / PostgreSQL) — job storage
    ↓
Hangfire server — in-process, or a separate worker service
    ↓
Job executed
    ↓
Hangfire Dashboard  /hangfire
```

---

## 5. The job types

```csharp
// fire and forget
BackgroundJob.Enqueue(() => emailService.SendWelcome(userId));

// delayed
BackgroundJob.Schedule(() => emailService.SendReminder(userId), TimeSpan.FromDays(1));

// recurring
RecurringJob.AddOrUpdate("nightly-cleanup", () => cleanup.Run(), Cron.Daily(3));

// continuation — runs after another job succeeds
BackgroundJob.ContinueJobWith(parentId, () => notify.Send(userId));
```

> [!TIP]
> `RecurringJob` is stored in the database and coordinated across servers, so **it runs once regardless of how many instances you deploy**. This solves the classic "cron fires on every replica" problem without a distributed lock.

---

## 6. Automatic retries

Hangfire retries failed jobs ten times by default, with increasing delays, and moves them to a **Failed** state afterwards — visible and re-queueable from the dashboard.

```csharp
[AutomaticRetry(Attempts = 5, DelaysInSeconds = new[] { 1, 5, 30, 120, 600 })]
public void SyncCrm(int orderId) { ... }
```

---

## 7. The dashboard

`/hangfire` gives you enqueued, processing, succeeded, failed and recurring jobs, with full exception details and a retry button.

> [!CAUTION]
> **The dashboard is unauthenticated by default and is only restricted to local requests.** Deploying it without an authorisation filter exposes job arguments — frequently containing personal data — and lets anyone trigger or delete jobs. This is the single most common Hangfire security mistake.

```csharp
app.UseHangfireDashboard("/hangfire", new DashboardOptions {
    Authorization = new[] { new AdminOnlyAuthorizationFilter() }
});
```

---

## 8. Real World Example

- **Enterprise .NET line-of-business applications** — the dominant use.
- **Nightly billing, report generation and data synchronisation.**
- **Small to medium systems** where adding RabbitMQ would be disproportionate.
- **Hangfire Pro** adds Redis storage and batching for higher throughput.

---

## 9. Communication and Dependencies

- **A database** — SQL Server or PostgreSQL, with Hangfire's own tables
- **A Hangfire server**, hosted inside the application or as a separate worker service
- **DI integration** — jobs resolve their dependencies from the container
- The dashboard, secured

---

## 10. The expression-tree gotcha

```csharp
BackgroundJob.Enqueue(() => service.Process(order));   // ✗ serialises the whole object
BackgroundJob.Enqueue(() => service.Process(orderId)); // ✓ serialises an int
```

> [!CAUTION]
> Hangfire serialises the method call, including its arguments. Passing an entity serialises the whole object into the job row — which is large, becomes stale, and **breaks every pending job when you change the class shape**. Pass identifiers.

---

## 11. Alternatives

```text
Hangfire        database-backed, dashboard, zero extra infrastructure
    ↓
Quartz.NET      powerful scheduling, no built-in dashboard
    ↓
MassTransit     full message bus over RabbitMQ or Azure Service Bus
    ↓
Azure Functions serverless background execution
    ↓
IHostedService  built into .NET — fine for simple in-process loops
```

> [!TIP]
> **MassTransit is the right step up** when you outgrow Hangfire: genuine message-bus semantics, routing, and cross-service messaging. Hangfire is a job runner, not a message bus.

---

## 12. When To Use / When NOT To Use

> [!TIP]
> Use Hangfire for .NET background jobs in small to medium systems where avoiding new infrastructure matters, and where transactional enqueue is valuable.

> [!CAUTION]
> - **Not for very high throughput** — the database becomes the bottleneck; polling storage does not match a broker's efficiency.
> - **Not for cross-service messaging** — it is a job runner within one application.
> - **Not for long durable workflows** — use Temporal or a workflow engine.

---

## 13. Advantages and Disadvantages

**Advantages**
- No additional infrastructure
- **Transactional enqueue** with your business data
- Dashboard included and genuinely good
- Recurring jobs coordinated across instances automatically
- Automatic retries with a visible failed state
- Simple, familiar C# API

**Disadvantages**
- Database polling adds load, and limits throughput
- Job arguments are serialised — a real versioning hazard
- Dashboard insecure by default
- In-process hosting means job work competes with request handling
- Redis storage and batching are commercial features

---

## 14. Performance Impact

| Aspect | Impact |
|---|---|
| **Throughput** | Hundreds to low thousands of jobs/minute — adequate for most systems |
| **Database load** | Polling and job-state writes are constant background traffic |
| **In-process hosting** | Jobs consume the same CPU as your web requests |
| **Retention** | Succeeded jobs are kept — configure expiry |

> [!TIP]
> **Host the Hangfire server in a separate worker service** in production, not inside the web application. Otherwise a CPU-heavy job degrades request latency, and scaling the web tier scales your job workers unintentionally.

---

## 15. Security Considerations

> [!CAUTION]
> Beyond the dashboard, the job storage itself is sensitive: **job arguments are stored in the database in plain form**, and a job carrying an email address or a document reference is personal data sitting in a table nobody thinks of as user data.

- **Secure the dashboard** with a real authorisation filter — not just IP restriction
- **Pass identifiers, never sensitive values**, as job arguments
- **Validate arguments** — an old queued job may carry a shape your current code does not expect
- **Least privilege** — the worker service's database user needs only what its jobs require
- **Set job expiry** so succeeded-job history does not accumulate indefinitely

---

## 16. Mental Model

> [!NOTE]
> **Hangfire is a to-do list written in the same notebook as your accounts.**
>
> There is no second system to keep in sync: if the accounts entry is torn out, the to-do goes with it. The trade is that a very long to-do list slows down the notebook you also use for everything else.

---

## 17. Mini Architecture Diagram

```text
ASP.NET Core API
    ↓ Enqueue (inside the same transaction)
Database: business tables + Hangfire tables
    ↓ polled by
Hangfire worker service (separate deployment)
    ↓
Job executed with DI-resolved dependencies
    ↓
Dashboard /hangfire (authenticated)
```

---

## 18. Complete Request Flow

```text
POST /orders
    ↓
BEGIN TRANSACTION
    order saved
    BackgroundJob.Enqueue(() => email.SendConfirmation(orderId))
COMMIT                       ← both, or neither
    ↓
201 returned
    ↓
Worker service polls storage and picks up the job
    ↓
Dependencies resolved from the DI container
    ↓
Loads order fresh by ID; already sent? → return   ← idempotency
    ↓
Email sent → job marked Succeeded
    ↓
─────────── failure ───────────
Exception → automatic retry with increasing delays
    ↓
Attempts exhausted → Failed state, visible in the dashboard with the stack trace
    ↓
Fix deployed → retry from the dashboard
```

---

## 19. Key Takeaway

> [!IMPORTANT]
> Hangfire gives .NET reliable background jobs with no new infrastructure and transactional enqueue — secure the dashboard, pass IDs rather than objects, and host the server separately from your web application.

---

## 20. Common Mistakes

- **Unsecured dashboard**, exposing job arguments and controls
- **Passing entities** instead of identifiers into the expression
- **Hosting the server in the web application** in production
- **No job expiry**, so succeeded-job history grows without limit
- **Expecting broker-level throughput** from database polling
- **Non-idempotent jobs**
- **Deploying a code change that breaks the signature** of jobs already queued

---

## 21. Open Source Technologies

- **Hangfire** (core is open source; Pro adds Redis storage and batches)
- **Quartz.NET** — scheduling-focused alternative
- **MassTransit**, **NServiceBus** — message bus frameworks
- **Coravel** — a lightweight alternative for small applications
- **Serilog + OpenTelemetry** — job observability

---

## 22. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 23. Workbook Exercise

- [ ] Check whether your Hangfire dashboard is reachable, and by whom.
- [ ] Look at your job arguments in storage — is any of it personal data?
- [ ] Verify that recurring jobs run once across all your instances, not once per instance.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
API → database (business + job tables) → Hangfire worker service → dashboard
```

## 2. Request Flow

```text
Input       a serialised method call enqueued, ideally in a transaction
    ↓
Processing  polled from storage, dependencies injected, retried on failure
    ↓
Output      job succeeded, or failed and visible in the dashboard
```

## 3. Real-World Usage

**Enterprise .NET applications** use Hangfire precisely because it adds no infrastructure. For a team already running SQL Server, a background job system that is one NuGet package and a set of tables is a very different proposition from deploying a broker.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A database-backed background job framework for .NET |
| **Why does it exist?** | So background processing needs no additional infrastructure |
| **Where does it belong?** | Beside your application, using the same database |
| **When should I use it?** | .NET background jobs at modest throughput — not as a message bus |
