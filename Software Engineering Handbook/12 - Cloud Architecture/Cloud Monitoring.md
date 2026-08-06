# Cloud Monitoring

> **In one line —** knowing what your system is doing when you cannot walk over to the machine; and the hard part is not collecting data, it is deciding what deserves to wake someone.

| | |
|---|---|
| **Category** | Concept / Practice |
| **Architectural Layer** | Observability |
| **Related notes** | [Monitoring and Alerting](../14%20-%20Scalability%20and%20Reliability/Monitoring%20and%20Alerting.md) · [Cloud Fundamentals](Cloud%20Fundamentals.md) · [AWS Architecture](AWS%20Architecture.md) · [Performance Engineering](../14%20-%20Scalability%20and%20Reliability/Performance%20Engineering.md) · [Cloud Security](Cloud%20Security.md) |

---

## 1. Short Definition

*What is it?*

Cloud monitoring is the collection of **metrics, logs, traces and events** from infrastructure you do not own, plus the alarms and dashboards built on them. On AWS the default toolset is CloudWatch, CloudTrail and X-Ray.

---

## 2. Problem

*What engineering problem does it solve?*

```text
On a server you owned
    ssh in, run top, tail the log, look at the disk
        ↓
In the cloud
    the instance was replaced twenty minutes ago
    the function ran for 300 ms and no longer exists
    there are forty containers and you do not know which served the request
    the database is managed and you cannot see its host at all
        ↓
Debugging by logging in is no longer possible
```

> [!IMPORTANT]
> **Ephemeral infrastructure forces telemetry to be shipped, not stored locally.** If a log line only exists on the instance that wrote it, you lose it exactly when you need it. This is why observability stopped being something you add after an incident and became a property the system must have before it goes live.

---

## 3. Architecture Position

```text
    APPLICATION            INFRASTRUCTURE          CONTROL PLANE
    logs, custom            CPU, memory,            who called
    metrics, traces         queue depth, errors      which API
         │                       │                       │
         └───────────┬───────────┴───────────┬───────────┘
                     ▼                       ▼
              CloudWatch                 CloudTrail
         (metrics, logs, alarms)      (audit, immutable)
                     │
        ┌────────────┼─────────────┐
        ▼            ▼             ▼
   dashboards     alarms       queries / insights
                     │
                     ▼
              a human, paged — only when action is required
```

---

## 4. The four kinds of telemetry

```text
METRICS    numbers over time, cheap, aggregated
           → "is it broken, and how badly"     → alarms live here

LOGS       discrete events with context, expensive at volume
           → "what exactly happened"           → investigation lives here

TRACES     one request's path across services, with timing per hop
           → "where did the time go"           → essential in distributed systems

EVENTS     state changes — deployments, scaling actions, API calls
           → "what changed just before this started"
```

> [!TIP]
> **Alert on metrics, investigate with logs, and diagnose with traces.** Teams that alert on log patterns end up with alarms that are slow, expensive and fragile. Teams with no traces spend incidents guessing which of eight services is slow. Each type has one job; using the wrong one for a job is most of what makes observability feel painful.

---

## 5. The AWS toolset, honestly

| Service | What it does | Verdict |
|---|---|---|
| **CloudWatch Metrics** | Infrastructure and custom metrics | Adequate; a coarse 1-minute default |
| **CloudWatch Alarms** | Thresholds and composite alarms | Fine, and correctly cheap |
| **CloudWatch Logs** | Ingestion, retention, Logs Insights | Works; **ingestion cost is the trap** |
| **CloudTrail** | Every API call, for audit | Non-negotiable, not really monitoring |
| **X-Ray** | Distributed tracing | Usable; OpenTelemetry is the better bet |
| **Container Insights** | ECS and EKS metrics | Convenient, and not cheap |
| **Synthetics / RUM** | Outside-in checks and real user data | Genuinely valuable, often skipped |

> [!CAUTION]
> **CloudWatch Logs bills on ingestion, and a single service left at debug level can exceed your compute bill.** Set retention on every log group — the default is "never expire" — and be deliberate about log level in production. This is the most common surprise line item in an AWS account after data transfer.

---

## 6. The metrics that actually matter

```text
THE FOUR GOLDEN SIGNALS
    LATENCY       and specifically the tail: p95, p99
    TRAFFIC       requests per second
    ERRORS        rate, split by cause
    SATURATION    how close a resource is to its limit

CLOUD-SPECIFIC ADDITIONS
    queue depth and message age      → work arriving faster than it drains
    database connection count        → the ceiling nobody watches
    Lambda concurrency and throttles → pressure downstream
    EBS burst balance, CPU credits   → invisible throttling
    API throttling / 429 rates       → provider quotas being hit
    COST                             → yes, this is an operational metric
```

> [!IMPORTANT]
> **Averages hide everything that matters. Percentiles are the only honest latency measurement.** An average of 100 ms with a p99 of 4 seconds means one request in a hundred is unusable — and at any real traffic level that is thousands of unhappy users a day. If you take one habit from this note, make it looking at p99 instead of mean.

---

## 7. Alerts: the discipline, not the tooling

```text
ALERT ON SYMPTOMS               NOT ON CAUSES
"error rate above 2%"           "CPU above 80%"
"p99 above 2 s"                 "memory at 90%"
"queue age above 5 min"         "disk at 70%"
    ↓                                ↓
users are affected               possibly nothing is wrong
```

```text
EVERY PAGE MUST PASS THREE TESTS
    1. is a user or the business affected?
    2. is there something a human can do about it right now?
    3. would you want to be woken for it?
    ↓
If any answer is no → it is a dashboard item or a ticket, not a page.
```

> [!CAUTION]
> **Alert fatigue is the actual failure mode of monitoring, and it is self-inflicted.** A channel with fifty daily notifications trains everyone to ignore all of them, including the real one. The fix is unpopular but simple: delete alarms that have never led to action. Fewer, better alarms beat comprehensive noise every time — and a monitoring system nobody trusts is worse than none, because it manufactures confidence.

---

## 8. Structured logs and correlation IDs

```text
UNSTRUCTURED                      STRUCTURED
"User 42 failed login"            {"level":"warn","event":"login_failed",
                                   "user_id":42,"request_id":"abc-123",
                                   "tenant":"acme","duration_ms":38}
    ↓                                  ↓
grep, and hope                    query, aggregate, alert
```

> [!TIP]
> **A request id propagated through every service and logged on every line is the cheapest observability win available.** With it, one incident becomes a single query returning the complete story across services. Without it, you are correlating timestamps by hand at two in the morning. Generate it at the edge, pass it in a header, include it in every log line, and return it to the client so a support ticket can be traced instantly.

---

## 9. Real World Example

- **An API's p99 latency alarm** — the single most useful alert most teams have.
- **Queue message age** — catching a stalled worker fleet before customers notice; see [AWS SQS](../10%20-%20Distributed%20Systems/AWS%20SQS.md).
- **RDS connection count and Performance Insights** — finding the query that saturated the database; see [RDS](RDS.md).
- **Lambda throttles and dead letter queue depth** — the two alarms every serverless system needs.
- **A synthetic canary** hitting the login flow every minute from outside — it catches DNS, certificate and CDN failures that internal metrics cannot see.
- **A cost anomaly alarm** — which has prevented more damage in more accounts than most technical alerts.

---

## 10. Communication and Dependencies

- **An agent or SDK** — CloudWatch agent, OpenTelemetry collector, or a vendor agent
- **IAM permissions** for whatever publishes telemetry; see [IAM](IAM.md)
- **Log retention settings** on every group, or the bill grows silently
- **A paging route** — SNS to PagerDuty or Opsgenie; an unread email is not an alert
- **Dashboards someone actually opens**, otherwise they are decoration
- **A runbook per alarm**, so being paged is actionable rather than alarming

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Start with four things on day one: the golden signals as metrics, structured logs with a request id, retention on every log group, and two or three alarms tied to user impact. That is a small amount of work and covers most incidents.

> [!CAUTION]
> - **Do not instrument everything before knowing what you would do with it** — cost grows faster than insight
> - **Do not page on causes** like CPU; page on symptoms
> - **Do not keep alarms that have never resulted in action**
> - **Do not log at debug level in production** unless you have priced the ingestion
> - **Do not build dashboards nobody reads** instead of alarms someone receives
> - **Do not rely only on internal metrics** — a synthetic check from outside catches what they cannot
> - **Do not put secrets or personal data in logs**; they are widely readable and long-lived

---

## 12. Advantages and Disadvantages

**Advantages**
- Visibility into infrastructure you cannot log in to
- Metrics for managed services you could not instrument yourself
- Alarms and auto scaling driven by the same data
- CloudTrail provides a complete, tamper-evident audit trail
- Correlating deployments with metric changes finds causes quickly
- Cost is observable as an operational signal

**Disadvantages**
- **Log ingestion cost scales with verbosity, not value**
- CloudWatch's interface and query language are unpleasant compared with alternatives
- Default 1-minute metric granularity misses short spikes
- Distributed tracing requires deliberate instrumentation across every service
- Very easy to build a wall of dashboards and still be blind
- Vendor tools are excellent and expensive; per-host pricing punishes elasticity
- No visibility below the provider's abstraction — you trust their metrics

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Metric publication** | Negligible; batch custom metrics rather than one call per event |
| **Log shipping** | Small CPU cost; use asynchronous, buffered writers |
| **Tracing** | Sample in production — 100% tracing at high volume is expensive |
| **Agents** | A few percent of CPU and memory per host |
| **Metric granularity** | High-resolution metrics cost more and are rarely needed |
| **Logs Insights queries** | Charged by data scanned — narrow the time range |

> [!TIP]
> **Sample traces, aggregate metrics, and log deliberately.** Head-based sampling at a few percent plus a rule that always keeps errors gives you nearly all the diagnostic value for a fraction of the volume. Emitting a metric per request and a log line per request at high traffic is how observability becomes a top-three line item.

---

## 14. Security Considerations

> [!CAUTION]
> **Logs are the most common accidental data leak in a cloud account.** Full request bodies, authorisation headers, tokens, card numbers and personal data end up in log groups that far more people can read than can read the database — and they are retained for years. Redact at the point of emission, not in a downstream pipeline that someone will bypass.

- **Never log secrets, tokens or personal data** — filter in the logger itself
- **CloudTrail to a separate account** with Object Lock, so an intruder cannot erase the record
- **Least privilege on log read access** — treat sensitive log groups like a database
- **Alert on security events too** — root usage, IAM policy changes, disabled logging, failed authentication spikes
- **Retention as a compliance decision**, not a default; both too short and too long carry risk
- **Encrypt log groups** holding regulated data
- **Monitoring is also an availability dependency** — decide what happens when it is the thing that is down

---

## 15. Mental Model

> [!NOTE]
> **Metrics are the dashboard of a car, logs are the black box recorder, traces are the route map, and alarms are the warning lights.**
>
> You glance at the dashboard while driving. You read the recorder after something happened. You consult the route map when the journey took longer than expected. And the warning lights are deliberately few — a car with forty blinking lights teaches you to ignore all of them, which is precisely what happens to teams with forty alerts.

---

## 16. Mini Architecture Diagram

```text
  Application (structured logs + request id + custom metrics + traces)
        │
        │  OpenTelemetry SDK / CloudWatch agent
        ▼
  ┌─────────────────────────────────────────────────┐
  │  METRICS          LOGS            TRACES        │
  │  1-min, cheap     retention SET   sampled       │
  └───────┬────────────────┬──────────────┬─────────┘
          │                │              │
     ┌────▼─────┐    ┌─────▼──────┐  ┌───▼────────┐
     │  ALARMS  │    │  Insights  │  │ service map│
     │ symptoms │    │  queries   │  │ latency by │
     │ only     │    │            │  │ hop        │
     └────┬─────┘    └────────────┘  └────────────┘
          │
      SNS → PagerDuty → the on-call engineer → a runbook
          │
   ┌──────▼───────────────────────────────────────┐
   │ Synthetic canary hits the real login flow    │
   │ from OUTSIDE, every minute                    │
   └──────────────────────────────────────────────┘

   CloudTrail ──────────► separate log-archive account (immutable)
```

---

## 17. Complete Request Flow

```text
Request arrives at the ALB → a request id is generated at the edge
    ↓
Passed as a header through every service; logged on every line
    ↓
Each service emits a trace span and a latency metric
    ↓
─────────────── the incident ───────────────
p99 latency alarm fires: above 2 s for 5 minutes  (a SYMPTOM)
    ↓
On-call is paged; the runbook links a dashboard
    ↓
Dashboard shows: traffic normal, error rate normal, p99 tripled
    ↓
Trace service map shows the time is spent in one service's database call
    ↓
RDS metrics: connection count at the ceiling, CPU low
    ↓
Performance Insights names the query — a deployment 40 minutes ago
removed an index usage by changing a filter
    ↓
Deployment event on the same timeline confirms the correlation
    ↓
Rolled back; p99 recovers; the alarm clears
    ↓
─────────────── afterwards ───────────────
A new alarm on database connection count, because it was the leading indicator
An unused alarm on instance memory is deleted, because it never once helped
Log retention on the noisiest group reduced from never to 14 days
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> Ship telemetry off the instance, alert on user-visible symptoms rather than causes, propagate a request id everywhere, watch p99 rather than averages, and delete every alarm that has never led to action.

---

## 19. Common Mistakes

- **No log retention set**, so ingestion and storage grow forever
- **Debug logging in production**, quietly outspending compute
- **Alerting on CPU and memory** instead of on user impact
- **Watching averages** while the p99 is catastrophic
- **Fifty alerts in one channel**, so nobody reads any of them
- **No request id**, making cross-service investigation manual
- **Unstructured logs** that can only be grepped
- **No traces** in a distributed system, so every incident starts with guessing
- **Secrets and personal data in logs**
- **CloudTrail in the audited account**, deletable by an intruder
- **Dashboards instead of alarms** — nobody is looking at 3 a.m.
- **No synthetic check**, so a DNS or certificate failure looks perfectly healthy from inside
- **No runbook attached to a page**, leaving the on-call engineer to improvise

---

## 20. Open Source Technologies

- **OpenTelemetry** — the vendor-neutral standard for metrics, logs and traces; instrument once
- **Prometheus** + **Grafana** — the default self-hosted stack, far better dashboards than CloudWatch
- **Loki**, **OpenSearch** — log aggregation and search
- **Jaeger**, **Tempo** — distributed tracing backends
- **Alertmanager** — routing, grouping and silencing, which CloudWatch handles poorly
- **Vector**, **Fluent Bit** — collect, transform and redact telemetry before it is shipped
- **node_exporter**, **cAdvisor** — host and container metrics
- **k6**, **Blackbox exporter** — synthetic checks from outside

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] List every alarm you have and mark which ones ever led to an action. Delete the rest.
- [ ] Check retention on every log group; set one on any that says "never expire".
- [ ] Confirm a request id flows from the edge through every service and into your logs.
- [ ] Replace one average-latency graph with p99 and see whether the story changes.
- [ ] Add one synthetic check that exercises your most important user flow from outside.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
app (structured logs, metrics, traces) → collector → metrics/logs/traces
                                              └→ alarms on symptoms → on-call + runbook
```

## 2. Request Flow

```text
Input       telemetry emitted by ephemeral infrastructure, shipped immediately
    ↓
Processing  metrics for alarms, logs for investigation, traces for diagnosis
    ↓
Output      a small number of pages that are always worth acting on
```

## 3. Real-World Usage

Mature teams look similar and unimpressive: a handful of symptom-based alarms, structured logs with a request id, sampled traces, one synthetic check, and log retention that is set. The immature version is not the one with less data — it is usually the one with far more data and no idea which number would tell them something is wrong.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Metrics, logs, traces and events shipped off infrastructure you cannot log in to |
| **Why does it exist?** | Because ephemeral, managed infrastructure cannot be debugged by hand |
| **Where does it belong?** | Beside everything, feeding a small set of alarms |
| **When should I use it?** | Before production — after the first incident is too late to instrument |
