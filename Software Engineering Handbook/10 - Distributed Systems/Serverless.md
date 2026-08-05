# Serverless

> **In one line —** you upload a function, the platform runs it on demand and charges only while it executes — and there are still servers, you just do not manage them.

| | |
|---|---|
| **Category** | Deployment Model |
| **Architectural Layer** | Infrastructure |
| **Also called** | FaaS — Functions as a Service |
| **Related notes** | [Lambda](../12%20-%20Cloud%20Architecture/Lambda.md) · [Cloud Fundamentals](../12%20-%20Cloud%20Architecture/Cloud%20Fundamentals.md) · [AWS SQS](AWS%20SQS.md) · [Microservices](Microservices.md) |

---

## 1. Short Definition

*What is it?*

Serverless means running code without provisioning or managing servers. The platform starts an instance when a request arrives, runs your function, and shuts it down. You pay per invocation and per millisecond of execution.

---

## 2. Purpose

*What is its main purpose?*

To eliminate capacity planning and idle cost. Nothing runs — and nothing is billed — when there is no traffic.

---

## 3. Problem

*What engineering problem does it solve?*

```text
TRADITIONAL                           SERVERLESS
provision servers for peak            no provisioning
pay 24/7 for capacity you use         pay per invocation
    at 9am                                ↓
patch, monitor, scale, replace        the platform handles all of it
    ↓                                     ↓
90% of the bill is idle capacity      zero cost when idle
```

---

## 4. Architecture Position

```text
Event source
 (HTTP · queue · schedule · file upload · database change)
        ↓
API Gateway / trigger
        ↓
┌──── FUNCTION ────┐
│  cold or warm    │
│  stateless       │
│  time-limited    │
└────────┬─────────┘
         ↓
Managed services: database, storage, queue
```

> [!IMPORTANT]
> **Serverless functions must be stateless.** Any instance may be destroyed after any invocation, and the next request may run on a completely new one. All state lives in a database, cache or object store.

---

## 5. Cold starts — the defining constraint

```text
No warm instance available
    ↓
Provision an execution environment
    ↓
Load the runtime
    ↓
Load your code and dependencies
    ↓
Initialise (database connections, framework boot)
    ↓
Finally: run the handler
```

| Runtime | Typical cold start |
|---|---|
| **Go, Rust** | ~10–100 ms |
| **Node.js, Python** | ~100–500 ms |
| **Java, .NET** | **1–3 seconds** |
| **Java with GraalVM / .NET AOT** | ~50–100 ms |

> [!CAUTION]
> **A large dependency tree is the main cause of slow cold starts.** A Python function importing pandas and boto3 can take seconds before your first line runs. Trim dependencies aggressively; it matters far more here than in a long-running service.

---

## 6. The database connection problem

```text
1,000 concurrent invocations
    ↓
1,000 function instances
    ↓
Each opens its own database connection
    ↓
PostgreSQL max_connections = 100
    ↓
"too many clients already"
```

> [!IMPORTANT]
> This is the single most common serverless failure, and it is structural: connection pooling assumes long-lived processes, and serverless has none. The answers are a **proxy** (RDS Proxy, PgBouncer), an **HTTP-based database** (Neon, PlanetScale, DynamoDB), or **not using serverless** for connection-heavy workloads.

---

## 7. Where it genuinely fits

```text
✓ Event processing            file uploaded → generate a thumbnail
✓ Scheduled jobs              nightly cleanup, report generation
✓ Webhooks                    infrequent, bursty, unpredictable
✓ Glue between services       small transformations
✓ Very spiky traffic          zero most of the time, huge occasionally
✓ Genuinely small APIs        a handful of endpoints, low volume

✗ Steady high traffic         a container is cheaper and faster
✗ Long-running work           15-minute execution limit on Lambda
✗ WebSockets and streaming    fundamentally a poor fit
✗ Latency-critical paths      cold starts are user-visible
✗ Heavy database use          the connection problem above
```

---

## 8. The cost curve

```text
Low, spiky traffic     serverless is dramatically cheaper — often near zero
Moderate traffic       roughly comparable
High steady traffic    serverless becomes SUBSTANTIALLY more expensive
```

> [!CAUTION]
> Serverless cost scales linearly with invocations, while a container's cost is flat. There is a crossover point, and past it the same workload can cost several times more. Model it against your real traffic pattern before committing an architecture to it.

---

## 9. Real World Example

- **Image processing on upload** — the archetypal fit: event-driven, bursty, short.
- **[SQS](AWS%20SQS.md) triggering Lambda** — background workers with no worker fleet to run.
- **Scheduled maintenance jobs** — no server needed for something that runs once a night.
- **Companies that migrated back to containers** after their traffic became steady and the bill grew — a well-documented pattern, not a rare one.

---

## 10. Beyond FaaS

"Serverless" now covers more than functions:

```text
Serverless functions     Lambda, Cloud Functions, Azure Functions
Serverless containers    AWS Fargate, Cloud Run       ← often the better middle ground
Serverless databases     Aurora Serverless, Neon, PlanetScale
Serverless everything    S3, SQS, DynamoDB — no servers were ever visible
```

> [!TIP]
> **Cloud Run and Fargate are frequently the right answer** when Lambda's constraints bite: an ordinary container, scaled to zero, with no 15-minute limit and no cold-start penalty on every scale-up. They are far less discussed than functions and fit more real workloads.

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Use serverless for event-driven, bursty, short-lived work where idle cost matters more than per-request cost — and where a few hundred milliseconds of occasional cold start is acceptable.

> [!CAUTION]
> Do not build an entire application as dozens of functions. That produces the debugging difficulty of [microservices](Microservices.md) plus cold starts plus vendor lock-in — and local development becomes genuinely painful. A container that scales to zero is usually the better trade.

---

## 12. Advantages and Disadvantages

**Advantages**
- No servers to provision, patch or scale
- Zero cost when idle
- Automatic scaling, including to very large bursts
- Fast to deploy a small piece of functionality
- Built-in availability across zones

**Disadvantages**
- **Cold starts**
- Execution time limits (15 minutes on Lambda)
- The database connection problem
- Vendor lock-in — triggers, permissions and packaging are platform-specific
- Local development and testing are harder
- Expensive at steady high volume
- Debugging is distributed by nature

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Warm invocation** | Comparable to a container |
| **Cold start** | 100 ms to 3 s, depending on runtime and dependencies |
| **Concurrency** | Scales automatically — sometimes faster than downstream systems can cope |
| **Memory** | Configurable, and CPU scales with it — more memory is often *cheaper* because it finishes sooner |
| **Timeouts** | A hard ceiling you must design around |

> [!TIP]
> **Increasing memory often reduces cost.** CPU allocation scales with memory, so a function given twice the memory may finish in less than half the time — and you are billed for duration.

---

## 14. Security Considerations

- **One IAM role per function**, with only the permissions that function needs — the fine granularity here is a genuine advantage over a monolithic role
- **Environment variables are visible** to anyone who can read the function configuration; use a secrets manager for real secrets
- **Dependencies are your attack surface**, and serverless encourages many small packages with many trees
- **Public function URLs** are internet-facing endpoints and need the same authentication as any API
- **Automatic scaling is also a billing attack surface** — an unauthenticated function can be invoked until it is expensive; set concurrency limits and rate limits
- **Cold instances may retain state in `/tmp`** between invocations on the same instance — never leave sensitive data there

---

## 15. Mental Model

> [!NOTE]
> **A server is a taxi you hire for the day; serverless is one you hail per trip.**
>
> If you make three trips a day, hailing is far cheaper. If you are driving continuously, hiring for the day wins easily. And the hailed taxi sometimes takes a minute to arrive — which is the cold start, and it is only a problem when someone is waiting.

---

## 16. Mini Architecture Diagram

```text
Event  (HTTP · queue · schedule · S3 upload)
    ↓
Trigger
    ↓
Function instance  ─ cold: provision + init + run
                   └ warm: run only
    ↓
RDS Proxy / HTTP database   ← the connection fix
    ↓
Managed storage · queue · database
```

---

## 17. Complete Request Flow

```text
Image uploaded to object storage
    ↓
Storage event triggers the function
    ↓
Warm instance available?
    ↓ NO  → cold start: environment + runtime + dependencies + init (~400 ms)
    ↓ YES → straight to the handler (~5 ms)
    ↓
Handler receives the event with the object key
    ↓
Connects via a proxy — not directly to the database
    ↓
Downloads, resizes, uploads the thumbnail
    ↓
Writes a record; publishes a completion event
    ↓
Returns; the instance stays warm for a while, then is destroyed
    ↓
Billed for exactly the milliseconds used
    ↓
10,000 uploads arrive at once → the platform runs 10,000 instances
    ↓
...which is when the database connection limit becomes the real constraint
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> Serverless removes capacity management and idle cost, and pays for it with cold starts, execution limits and the database connection problem — it suits bursty event-driven work, not steady traffic.

---

## 19. Common Mistakes

- **Direct database connections** without a proxy, exhausting the connection limit
- **Large dependency bundles**, causing multi-second cold starts
- **Building a whole application** as dozens of functions
- **Ignoring the cost curve** as traffic becomes steady
- **Assuming state persists** between invocations
- **No concurrency limit**, allowing runaway cost
- **Secrets in environment variables**
- **Choosing Lambda where a container that scales to zero would fit better**

---

## 20. Open Source Technologies

- **AWS Lambda**, **Google Cloud Functions**, **Azure Functions**, **Cloudflare Workers**
- **AWS Fargate**, **Google Cloud Run**, **Knative** — serverless containers
- **Serverless Framework**, **AWS SAM**, **SST** — deployment tooling
- **LocalStack**, **`sam local`** — local development
- **RDS Proxy**, **PgBouncer**, **Neon**, **PlanetScale** — the connection problem
- **OpenFaaS**, **Fission** — self-hosted FaaS on Kubernetes

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Measure your function's cold start and work out how much of it is dependency loading.
- [ ] Model your monthly cost on serverless versus a small container at your actual traffic.
- [ ] Check what happens to your database when 500 function instances start at once.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Event → trigger → function instance (cold or warm) → proxy → managed services
```

## 2. Request Flow

```text
Input       an event from HTTP, a queue, a schedule or a storage change
    ↓
Processing  an instance is started if needed, the handler runs, the instance is discarded
    ↓
Output      a result, billed by the millisecond, with no idle cost
```

## 3. Real-World Usage

**Image processing triggered by an upload** is the pattern serverless fits best: unpredictable, bursty, short-lived, and idle most of the time. It is also the pattern that makes the cost argument obvious — a container waiting for uploads is billed continuously.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Running code on demand with no servers to manage |
| **Why does it exist?** | To remove capacity planning and the cost of idle infrastructure |
| **Where does it belong?** | Between event sources and managed backing services |
| **When should I use it?** | Bursty, event-driven, short work — not steady traffic or long jobs |
