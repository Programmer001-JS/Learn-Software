# Lambda

> **In one line —** you upload a function and AWS runs it on demand, scaling from zero to thousands of copies — as long as you accept that every invocation is stateless, brief, and cold sometimes.

| | |
|---|---|
| **Full name** | AWS Lambda |
| **Category** | FaaS — Function as a Service |
| **Architectural Layer** | Application |
| **Related notes** | [Serverless](../10%20-%20Distributed%20Systems/Serverless.md) · [AWS Architecture](AWS%20Architecture.md) · [EC2](EC2.md) · [AWS SQS](../10%20-%20Distributed%20Systems/AWS%20SQS.md) · [S3](S3.md) · [IaaS PaaS SaaS](IaaS%20PaaS%20SaaS.md) |

---

## 1. Short Definition

*What is it?*

Lambda runs a **function in response to an event**. There is no server to provision, no process to keep alive, and no scaling to configure. You are billed for the milliseconds your code actually executes.

---

## 2. Problem

*What engineering problem does it solve?*

```text
A task that runs 200 times a day, for 300 ms each time
    ↓
Total compute: 60 seconds per day
    ↓
On an instance: pay for 86,400 seconds, patch the OS, monitor the process
    ↓
99.93% of what you pay for is idle
```

Lambda inverts the unit of billing from **time held** to **work done**. For intermittent, event-shaped work, that is a change of two or three orders of magnitude.

> [!IMPORTANT]
> **The bigger win is not the bill — it is that scaling stops being a design problem.** One event or ten thousand simultaneous events are handled the same way, with no Auto Scaling Group, no warm-up time and no capacity planning. For genuinely spiky load, that elasticity is difficult to match with anything you operate yourself.

---

## 3. Architecture Position

```text
EVENT SOURCES                        LAMBDA                    TARGETS
─────────────                        ──────                    ───────
API Gateway / ALB / URL  ─────┐
S3 object created        ─────┤
SQS message              ─────┼──►  your handler  ──────►  RDS (via proxy)
EventBridge schedule     ─────┤     (stateless,           DynamoDB
DynamoDB stream          ─────┤      15 min max)          S3
Kinesis / MSK            ─────┤                            SQS / SNS
another Lambda           ─────┘                            an HTTP API
```

Lambda runs **outside your VPC by default**, which is why it can reach S3 and DynamoDB but not your database until you attach it to your subnets.

---

## 4. The execution model

```text
COLD START
    a request arrives, no warm environment exists
        ↓
    download the code → start the runtime → run init code → run the handler
        ↓
    100 ms to several seconds, depending on runtime and package size

WARM INVOCATION
    the environment is reused
        ↓
    run the handler only — single-digit milliseconds of overhead

CONCURRENCY
    one environment handles ONE request at a time
    100 simultaneous requests → 100 environments → 100 cold starts
```

> [!TIP]
> **Everything above the handler runs once per environment, not once per request.** Put database connections, SDK clients and configuration loading in the module scope and they are reused across warm invocations. This single habit is the difference between a fast Lambda and a slow one, and it is the most commonly missed optimisation.

> [!CAUTION]
> **Concurrency, not requests per second, is the number that matters.** Ten requests per second each taking two seconds means twenty concurrent environments — and every one of them holds a database connection. This is how a Lambda function quietly exhausts an RDS instance that was fine yesterday.

---

## 5. The hard limits

```text
15 MINUTES        maximum execution time, no exceptions
10 GB             maximum memory — and CPU scales WITH memory
512 MB - 10 GB    /tmp scratch space, ephemeral
6 MB              synchronous request/response payload
250 MB            unzipped deployment package (10 GB for container images)
1,000             default concurrent executions per account — raisable
```

> [!TIP]
> **Memory is the CPU dial, and the cheapest optimisation available.** Doubling memory roughly doubles CPU, so a function may finish in a third of the time for double the per-millisecond price — a net saving. Tune it with a tool like AWS Lambda Power Tuning rather than leaving everything at 128 MB and assuming that is thrifty.

> [!CAUTION]
> **The 15-minute ceiling is architectural, not a configuration you can argue with.** If a job might exceed it, Lambda is the wrong home — either split the work into a queue-driven chain, or run it on Fargate or EC2. Discovering this limit halfway through building a video encoder is a well-worn path.

---

## 6. Synchronous, asynchronous, and stream

```text
SYNCHRONOUS (API Gateway, ALB, direct invoke)
    caller waits → errors returned to the caller → NO automatic retry

ASYNCHRONOUS (S3, SNS, EventBridge)
    event queued → retried twice automatically → then to a dead letter queue

STREAM / POLL (SQS, Kinesis, DynamoDB streams)
    Lambda polls in batches → a failure retries the BATCH
    → a poison message can block a partition until it expires
```

> [!CAUTION]
> **Retries mean your function must be idempotent, and this is not optional.** The same event will be delivered twice — by a retry, a batch failure, or an at-least-once source. If handling it twice charges a card twice or sends two emails, you have a bug that will appear in production and be very hard to reproduce. Deduplicate on an event id.

---

## 7. Lambda and the VPC

```text
DEFAULT              in AWS's network
    → can reach S3, DynamoDB, public APIs
    → CANNOT reach RDS in a private subnet

ATTACHED TO A VPC    an elastic network interface in your subnets
    → can reach RDS and internal services
    → loses internet access unless a NAT Gateway exists
    → NAT Gateway costs per hour AND per GB
```

> [!TIP]
> **VPC-attached Lambda is no longer slow to start, but it is still a cost decision.** The old multi-second ENI penalty is gone. What remains is that outbound internet now needs a NAT Gateway, and that your function competes for the same database connections as everything else. Use RDS Proxy so hundreds of concurrent functions share a bounded pool.

---

## 8. Where Lambda genuinely fits

```text
EXCELLENT                              POOR
event handlers (S3, SQS, streams)      steady high-traffic APIs
scheduled jobs and cron                long-running jobs (>15 min)
glue between AWS services              latency-critical paths (p99 matters)
webhook receivers                      anything needing WebSockets held open
image/thumbnail processing             heavy CPU work at constant volume
low-traffic or internal APIs           workloads with big warm caches
spiky, unpredictable load              tight loops calling other services
```

> [!IMPORTANT]
> **The crossover point is roughly "constant, predictable traffic".** At steady volume a container on Fargate or an EC2 instance is cheaper, faster at the tail, and easier to debug. Lambda's economics come from idleness — if your function is never idle, you are paying a premium for elasticity you are not using.

---

## 9. Real World Example

- **S3 upload triggers** — thumbnails, virus scanning, metadata extraction.
- **Scheduled maintenance** — nightly reports, cleanup, cache warming, replacing cron on a forgotten server.
- **Webhook endpoints** — Stripe, GitHub, third-party callbacks that arrive unpredictably.
- **Queue workers** — SQS to Lambda, scaling with the queue depth; see [AWS SQS](../10%20-%20Distributed%20Systems/AWS%20SQS.md).
- **Glue and automation** — reacting to CloudWatch alarms or tagging new resources.
- **Small internal APIs** where cold starts are irrelevant and the traffic is negligible.

---

## 10. Communication and Dependencies

- **An event source** — Lambda does nothing on its own
- **An IAM execution role** — least privilege, per function
- **RDS Proxy** if it touches a relational database
- **A dead letter queue** for asynchronous invocations, or failures vanish silently
- **CloudWatch Logs** — the only real window into what happened
- **Secrets Manager or Parameter Store**, loaded in the init phase and cached

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Use Lambda for event-driven work, scheduled tasks, glue between services, and anything whose traffic is spiky or occasional. It is also the right first answer for a job you would otherwise put in a cron entry on a server nobody maintains.

> [!CAUTION]
> - **Not for anything over 15 minutes** — the limit is absolute
> - **Not for steady high-traffic APIs** — containers cost less and behave better at p99
> - **Not where cold-start latency is user-visible and unacceptable**
> - **Not for stateful or long-lived connections** — no WebSockets, no in-process cache you can rely on
> - **Not as a distributed monolith** — dozens of functions calling each other synchronously is harder to operate than one service
> - **Not for chatty database work** — connection pressure and per-call latency both work against you

---

## 12. Advantages and Disadvantages

**Advantages**
- No servers, no patching, no capacity planning
- Scales from zero to very high concurrency automatically
- Billed per millisecond of actual execution
- Costs nothing at rest
- Native integration with most AWS event sources
- Built-in retries and dead-letter handling for async work
- Excellent for small, isolated units of work

**Disadvantages**
- **Cold starts**, which land on real users at the tail
- **A hard 15-minute limit** and a hard memory ceiling
- Expensive at sustained high volume
- Local development and debugging are worse than for a normal process
- Distributed tracing is essential and still awkward
- Strong lock-in through the event model
- Stateless by force — every invocation reconnects to something
- Easy to sprawl into hundreds of functions nobody can map

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Cold start** | ~100–300 ms for Node/Python, seconds for a large JVM or .NET package |
| **Warm invocation** | A few milliseconds of overhead |
| **Memory** | CPU scales with it — more memory often costs less overall |
| **Provisioned concurrency** | Removes cold starts, at the price of paying for idle |
| **SnapStart** | Restores a pre-initialised snapshot; a large win for Java |
| **Package size** | Directly affects cold start — trim dependencies and use layers |
| **Concurrency** | Every concurrent execution is a separate connection to everything downstream |

> [!TIP]
> **Fix cold starts by shrinking the package and moving work into init before you pay for provisioned concurrency.** Most functions that "need" provisioned concurrency simply import an entire SDK and open a connection per request. Provisioned concurrency is a real tool, but it also reintroduces the thing you came here to avoid: paying for idle capacity.

---

## 14. Security Considerations

> [!CAUTION]
> **One IAM role per function, scoped to exactly what that function does.** The default temptation — a single shared role with broad permissions across all functions — means compromising the most trivial webhook handler grants an attacker everything every function can do. Functions are small; their permissions should be too.

- **No secrets in environment variables in plaintext** — they are visible to anyone with console read access; use Secrets Manager and cache the value in init
- **Least privilege on the execution role**, and separate roles per function
- **Validate every event payload** — webhook and API events are untrusted input
- **Dependencies are your responsibility**, not AWS's — scan them
- **`/tmp` persists across warm invocations** — never leave sensitive data there
- **Set a concurrency limit** on internet-facing functions; without one, an attack becomes an unbounded bill
- **Beware the confused deputy** — verify the source of cross-account and third-party invocations

---

## 15. Mental Model

> [!NOTE]
> **Lambda is a vending machine, not a chef.**
>
> Insert an event, get a result, pay only for the item. The machine is idle at no cost, and a hundred people can be served at once because there are a hundred machines. But it cannot cook a three-course meal, it forgets you the moment you walk away, and the first item of the morning takes a little longer to drop. Nobody runs a restaurant on vending machines — and nobody should install a chef to dispense one bottle of water a day.

---

## 16. Mini Architecture Diagram

```text
   S3 upload      SQS queue      EventBridge (cron)      API Gateway
       │              │                  │                    │
       └──────────────┴────────┬─────────┴────────────────────┘
                               ▼
                ┌──────────── LAMBDA ─────────────┐
                │  init  (runs once per env)      │
                │    clients, secrets, pool       │
                │  ─────────────────────────────  │
                │  handler (runs per event)       │
                │    idempotent, ≤15 min          │
                │  IAM role: this function only   │
                └───────┬─────────────────┬───────┘
                        │                 │
              in-VPC ENI│                 │ no VPC needed
                        ▼                 ▼
                 RDS Proxy → RDS      S3 · DynamoDB · SQS
                        │
             failures ──▼──► Dead Letter Queue → alarm
```

---

## 17. Complete Request Flow

```text
A 200 MB video is uploaded to S3
    ↓
S3 emits ObjectCreated → Lambda invoked asynchronously
    ↓
No warm environment exists → COLD START
    code downloaded, runtime started, init runs:
    SDK clients created, secret fetched, connection pool opened
    ↓
Handler runs: reads metadata, writes a row via RDS Proxy,
enqueues a transcode job on SQS  (400 ms)
    ↓
Environment stays warm — the next 50 events skip init entirely
    ↓
─────────────── burst ───────────────
500 files uploaded at once
    ↓
Lambda starts ~500 environments; most are cold
    ↓
Reserved concurrency caps it at 100, protecting the database
    ↓
The rest are retried automatically by the async invocation queue
    ↓
─────────────── failure ───────────────
One event fails three times (a malformed file)
    ↓
Sent to the dead letter queue → CloudWatch alarm → a human looks at it
    ↓
The other 499 completed unaffected
    ↓
─────────────── duplicate ───────────────
S3 delivers one event twice
    ↓
The handler checks the event id against a processed-events table → no-op
    ↓
No duplicate charge, no duplicate email — because it was written idempotently
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> Lambda is excellent for event-driven, spiky and occasional work — write handlers idempotently, do setup in init, cap concurrency to protect what is downstream, and move to containers once traffic becomes steady.

---

## 19. Common Mistakes

- **Opening a database connection inside the handler** instead of in init
- **No RDS Proxy**, so concurrency exhausts the database's connections
- **Assuming exactly-once delivery** and not writing idempotent handlers
- **No dead letter queue**, so asynchronous failures disappear silently
- **Leaving everything at 128 MB**, making functions slow and often more expensive
- **Using it for jobs that grow past 15 minutes**, then rewriting under pressure
- **Plaintext secrets in environment variables**
- **One shared, over-permissive execution role** for every function
- **No reserved concurrency**, so a traffic spike becomes an unbounded bill
- **A distributed monolith** — dozens of functions calling each other synchronously
- **Choosing Lambda for a steady, high-traffic API** and paying a premium for elasticity nobody uses

---

## 20. Open Source Technologies

- **AWS SAM**, **Serverless Framework**, **AWS CDK** — define and deploy functions
- **Terraform / OpenTofu** — if your infrastructure already lives there
- **LocalStack**, **SAM local** — invoke functions locally
- **Powertools for AWS Lambda** — logging, tracing, idempotency, batch handling; genuinely worth adopting
- **AWS Lambda Power Tuning** — finds the memory setting with the best cost/latency
- **OpenTelemetry** — tracing across functions, which you will need
- **Knative**, **OpenFaaS** — the same model without the provider

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Take one function and move every client and secret load out of the handler into init. Measure the difference.
- [ ] Ask of each function: what happens if this event arrives twice? Fix any that answer badly.
- [ ] Check that every asynchronous function has a dead letter queue with an alarm on it.
- [ ] Pick your busiest function and compare its monthly cost against one small Fargate task.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
event source → Lambda (init once, handler per event) → data stores
                              └── failures → DLQ → alarm
```

## 2. Request Flow

```text
Input       an event from a source you configured
    ↓
Processing  a warm environment if one exists, otherwise a cold start; one event at a time
    ↓
Output      a result or a retry — billed per millisecond, scaled without your involvement
```

## 3. Real-World Usage

Lambda's durable role turned out not to be "the future of all applications" but something narrower and genuinely valuable: **the glue and the reflexes of a cloud system.** Uploads trigger it, queues drain into it, schedules fire it, alarms wake it. Steady request-serving drifted back to containers, and the two now coexist in most mature AWS architectures.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A function AWS runs on demand, billed per millisecond |
| **Why does it exist?** | Because intermittent work should not pay for idle servers |
| **Where does it belong?** | Between event sources and data stores — the glue of the architecture |
| **When should I use it?** | Event-driven, scheduled or spiky work under 15 minutes |
