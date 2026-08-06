# Monitoring and Alerting

> **In one line —** deciding what "working" means, measuring it, and waking a human only when it is not; the hard part is not the tooling, it is having the discipline to delete alerts.

| | |
|---|---|
| **Category** | Concept / Practice |
| **Architectural Layer** | Observability |
| **Related notes** | [Cloud Monitoring](../12%20-%20Cloud%20Architecture/Cloud%20Monitoring.md) · [High Availability](High%20Availability.md) · [Performance Engineering](Performance%20Engineering.md) · [Load Testing](Load%20Testing.md) · [Deployment Strategies](../13%20-%20DevOps%20and%20Delivery/Deployment%20Strategies.md) |

---

## 1. Short Definition

*What is it?*

Monitoring is measuring whether the service is doing its job. Alerting is the much narrower discipline of **deciding which measurements justify interrupting a person**. The second is where teams succeed or fail.

> **Scope note:** [Cloud Monitoring](../12%20-%20Cloud%20Architecture/Cloud%20Monitoring.md) covers the mechanics — metrics, logs, traces, CloudWatch. This note covers the practice: SLIs, SLOs, error budgets, on-call, and incident response.

---

## 2. Problem

*What engineering problem does it solve?*

```text
Without it
    users tell you the site is down, via Twitter
    "is it slow?" is answered by opinion
    an incident is diagnosed by guessing and restarting things
    ↓
With it done BADLY
    400 alerts a day
    everyone mutes the channel
    the one real alert arrives in the middle of the noise and is missed
    ↓
Bad monitoring is worse than none: it manufactures confidence
```

> [!IMPORTANT]
> **The failure mode of monitoring is not missing data — it is alert fatigue.** Every organisation that has been burned once responds by adding alerts, and the result is a channel nobody reads. The mature move is subtractive: define what "working" means, alert on that, and delete everything that has never led to an action. Fewer, trusted alerts beat comprehensive noise, every time.

---

## 3. Architecture Position

```text
   SLI          what we measure          "successful requests / total requests"
    │
   SLO          the target                "99.9% over 30 days"
    │
   ERROR BUDGET what we may spend         "43 minutes of failure this month"
    │
   ALERT        fires when the budget is burning too fast
    │
   ON-CALL      a human, with a runbook
    │
   INCIDENT     mitigate → resolve → review → change something
```

---

## 4. SLI, SLO, SLA

```text
SLI   Service Level INDICATOR    a measurement
      → availability: fraction of requests that succeeded
      → latency: fraction of requests faster than 300 ms

SLO   Service Level OBJECTIVE    your internal target
      → 99.9% of requests succeed, measured over 30 days

SLA   Service Level AGREEMENT    a contract, with money attached
      → always LOOSER than your SLO, because you want warning
```

> [!TIP]
> **Define SLIs from the user's perspective, not the server's.** "CPU below 80%" is not an SLI — a user does not care. "99.9% of checkout requests complete successfully in under 2 seconds" is, because it describes the experience you are actually selling. Measure it as close to the user as possible: at the load balancer or, better, with a synthetic check from outside.

---

## 5. Error budgets: the idea that changes behaviour

```text
SLO 99.9% over 30 days  →  ERROR BUDGET = 43 minutes of failure

BUDGET REMAINING          → ship freely, take risks, deploy on Friday
BUDGET EXHAUSTED          → stop feature work, fix reliability
```

> [!IMPORTANT]
> **The error budget resolves the permanent argument between "ship faster" and "be more reliable" by turning it into arithmetic.** 100% is the wrong target: it is unachievable, and pursuing it means never changing anything. An explicit budget says that some failure is acceptable and *spending it deliberately is allowed* — which gives engineers permission to move and gives reliability work an objective trigger rather than depending on who argues most persuasively.

---

## 6. What to alert on

```text
ALERT ON SYMPTOMS — things a user experiences
    error rate above the SLO burn threshold
    p99 latency above the SLO
    a queue's oldest message older than N minutes
    a synthetic check of the login flow failing
    a certificate expiring in 14 days
    a backup job that has not succeeded

DO NOT PAGE ON CAUSES
    CPU at 85%          → so what? is anyone affected?
    memory at 90%       → that may be entirely normal
    a single pod restart → the orchestrator handled it
    disk at 70%         → predict, do not page
```

```text
THREE TESTS FOR EVERY PAGE
    1. is a user or the business affected?
    2. can a human do something about it right now?
    3. would you want to be woken up for this?
   Any "no" → dashboard or ticket, not a page.
```

> [!TIP]
> **Alert on the *rate* at which you are burning error budget, not on a fixed threshold.** A burn-rate alert distinguishes "we will exhaust a month's budget in an hour" (page immediately) from "we are slightly over target" (a ticket). This single technique removes most false pages while catching real incidents faster — a fixed 1% error threshold is simultaneously too twitchy at 3 a.m. and too slow during a serious outage.

---

## 7. On-call that people can sustain

```text
WHAT MAKES IT WORK
    every page has a RUNBOOK linked from the alert
    a rotation large enough that it is not one person's life
    handover with context, not just a schedule
    time to fix the causes of pages — compensated, not squeezed in
    the person on call has the ACCESS to fix things

WHAT DESTROYS IT
    pages that require no action
    pages at 3 a.m. that could have waited until 9
    no runbook, so every incident starts from first principles
    no authority to change anything
    the same alert every night for a month
```

> [!CAUTION]
> **The number of pages per shift is a reliability metric, and it should be trending down.** More than one or two actionable pages per week is a signal that the system — or the alerting — needs work, not that the on-call engineer needs resilience. Teams that treat on-call load as a fact of life rather than a bug end up with attrition instead of reliability.

---

## 8. Incidents: mitigate first, understand later

```text
DURING
    1. MITIGATE — restore service. Roll back. Fail over. Shed load.
       → do NOT debug root cause while users are affected
    2. COMMUNICATE — a status page, and internal updates on a cadence
    3. ONE incident commander, whose job is coordination, not fixing

AFTER
    a BLAMELESS review: timeline, contributing factors, what we change
    → the output is ACTIONS with owners, not a narrative
```

> [!IMPORTANT]
> **Rolling back before understanding is almost always correct.** The instinct to diagnose first is strong and expensive: every minute spent understanding is a minute of user impact, and the evidence will still be there afterwards. Restore service, then investigate with the logs, traces and the artefact you rolled back from. The exception is a data-corrupting bug, where continuing may be worse than stopping.

> [!TIP]
> **"Blameless" does not mean "no cause" — it means the cause is never a person.** If a single engineer's mistake could break production, the system permitted it: no review, no staging, no canary, no rollback. Those are the findings worth writing down.

---

## 9. Real World Example

- **A p99 latency SLO on the checkout flow**, with a burn-rate alert — the single most useful alert most teams have.
- **Queue message age**, catching a stalled worker fleet before customers notice.
- **A synthetic canary** logging in every minute from outside, which catches DNS, TLS and CDN failures that internal metrics cannot see.
- **Certificate expiry alerts at 30 and 14 days** — a startlingly common cause of self-inflicted outages.
- **Backup job failure alerts**, since a silently broken backup job is invisible for months; see [Backup Strategy](Backup%20Strategy.md).
- **Cost anomaly alerts**, which have prevented more damage in more accounts than most technical alerts.

---

## 10. Communication and Dependencies

- **Instrumented applications** — structured logs, metrics, traces; see [Cloud Monitoring](../12%20-%20Cloud%20Architecture/Cloud%20Monitoring.md)
- **A paging system** that actually reaches a phone — email is not alerting
- **A runbook per alert**, linked from the alert itself
- **A status page hosted outside your infrastructure**
- **Synthetic checks from outside your network**
- **An on-call rotation with agreed expectations**
- **Time allocated to act on review findings**, or the same incident recurs

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Start with four alerts: availability, latency, a queue or saturation signal, and an external synthetic check. Add a runbook to each. That covers most real incidents and can be built in a day.

> [!CAUTION]
> - **Do not alert on causes** — CPU, memory and disk belong on dashboards
> - **Do not page for anything that can wait until morning** — use a ticket for those
> - **Do not keep an alert that has never led to an action** — delete it
> - **Do not target 100% availability** — it prevents change and cannot be met
> - **Do not rely on dashboards** for detection; nobody is looking at 3 a.m.
> - **Do not build monitoring that depends on the system it monitors**
> - **Do not run on-call without a runbook and without access**

---

## 12. Advantages and Disadvantages

**Advantages**
- Problems found before users report them
- SLOs make reliability a measurable, negotiable property
- Error budgets end an unwinnable argument with arithmetic
- Incidents get shorter as runbooks accumulate
- Reviews turn each failure into a permanent improvement
- Data replaces opinion in performance discussions

**Disadvantages**
- **Alert fatigue is the default outcome** without active pruning
- Instrumentation and log volume cost real money
- On-call has a human cost that must be compensated
- SLOs require agreement across engineering and the business, which is slow
- Poorly chosen SLIs measure the wrong thing convincingly
- Monitoring is itself a system that can fail

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Metrics** | Negligible when aggregated; expensive if per-request and high cardinality |
| **Logging** | CPU and I/O cost, and ingestion billing that scales with verbosity |
| **Tracing** | Sample in production; always keep errors |
| **High cardinality labels** | The main cause of a metrics system falling over |
| **Alert evaluation** | Cheap; complex multi-window queries less so |
| **Synthetic checks** | Trivial load, disproportionate value |

> [!CAUTION]
> **High-cardinality labels are how a metrics system is destroyed.** A label containing a user id, a request id or a full URL path creates a separate time series per value — millions of them — and Prometheus or your vendor bill will not survive it. Identifiers belong in logs and traces; metrics take bounded labels only.

---

## 14. Security Considerations

> [!CAUTION]
> **Logs are the most common accidental data leak in a production system.** Request bodies, authorisation headers, tokens and personal data end up in log storage that far more people can read than can read the database, retained for years. Redact at the point of emission, in the logger, not in a downstream pipeline someone will eventually bypass.

- **Never log secrets, tokens or personal data** — filter at source
- **Alert on security signals too** — root account use, IAM policy changes, disabled logging, authentication failure spikes
- **Audit logs in a separate account**, so an intruder cannot erase the evidence
- **Restrict log read access** — treat sensitive log groups like a database
- **Health and status endpoints must not leak** versions, dependencies or stack traces
- **Your monitoring system is a target** — it knows your architecture and often holds credentials
- **Paging systems must be reachable during an outage**, including if your identity provider is the failure

---

## 15. Mental Model

> [!NOTE]
> **Monitoring is the instrument panel; alerting is the small set of warning lights.**
>
> The panel has dozens of readings you consult when something feels wrong. The warning lights are deliberately few, because a car with forty flashing lights teaches you to ignore all of them — which is precisely what happens to a team with forty alerts. And an error budget is the fuel gauge: it tells you how much margin remains, so you can drive briskly while you have it and slow down when you do not.

---

## 16. Mini Architecture Diagram

```text
   USER PERSPECTIVE                    SYSTEM PERSPECTIVE
   synthetic check (outside)           metrics · logs · traces
        │                                       │
        └──────────────┬────────────────────────┘
                       ▼
        ┌──────────────────────────────────┐
        │  SLIs                             │
        │   availability: success / total   │
        │   latency: % under 300 ms         │
        └──────────────┬───────────────────┘
                       ▼
        ┌──────────────────────────────────┐
        │  SLO: 99.9% / 30 days             │
        │  ERROR BUDGET: 43 min             │
        └──────────────┬───────────────────┘
                       ▼
        ┌──────────────────────────────────┐
        │  BURN-RATE ALERTS                 │
        │   fast burn (14×) → PAGE          │
        │   slow burn (2×)  → ticket        │
        └──────────────┬───────────────────┘
                       ▼
        on-call engineer ──► runbook linked from the alert
                       │
                       ├──► MITIGATE (roll back / fail over)
                       ├──► status page (hosted elsewhere)
                       └──► blameless review → actions with owners

   DASHBOARDS: CPU, memory, disk, per-instance detail
               ← for investigation, NEVER for paging
```

---

## 17. Complete Request Flow

```text
─────────────── defining it ───────────────
The team agrees: 99.9% of checkout requests succeed in under 2 s, over 30 days
    ↓
Error budget: 43 minutes of failure per month
    ↓
Two alerts configured: fast burn (pages) and slow burn (ticket)
    ↓
A synthetic check exercises the real login and checkout flow every minute
    ↓
─────────────── an incident ───────────────
02:14 — fast-burn alert: error rate would exhaust the month's budget in 40 minutes
    ↓
On-call paged. The alert links a runbook.
    ↓
Runbook step 1: was there a recent deployment? Yes — 02:05.
    ↓
MITIGATE FIRST: roll back. Service normal by 02:21.
    ↓
Total impact: 7 minutes. Budget spent: 7 of 43 minutes.
    ↓
No debugging happened during the incident — deliberately
    ↓
─────────────── the review ───────────────
Traces show the release added an uncached query on the checkout path
    ↓
Blameless findings:
    the canary ran for 5 minutes; this took 8 to manifest → extend the window
    load testing did not cover checkout under realistic cache state
    the runbook worked, and shortened the incident by an estimated 15 minutes
    ↓
Three actions, three owners, dates
    ↓
─────────────── the budget conversation ───────────────
Mid-month, 38 of 43 minutes are spent
    ↓
Feature deployments pause; the team works on reliability instead
    ↓
No argument was needed, because the rule was agreed in advance
    ↓
─────────────── pruning ───────────────
Quarterly review of alerts: 6 of 19 have never led to an action
    ↓
Deleted. The channel becomes trustworthy again.
    ↓
─────────────── what the alerts caught that nothing else would ───────────────
A certificate expiring in 14 days
A backup job silently failing for 3 weeks
An external synthetic check failing while every internal metric looked perfect
    (a DNS change had broken resolution for real users only)
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> Define what "working" means from the user's point of view, alert on burn rate against that objective, attach a runbook to every page, mitigate before diagnosing, and delete every alert that has never led to an action.

---

## 19. Common Mistakes

- **Paging on causes** — CPU, memory, disk — instead of user impact
- **Alert fatigue tolerated**, so the real alert arrives inside the noise
- **No runbook**, so every incident starts from first principles at 3 a.m.
- **Fixed thresholds instead of burn rate**, producing both false pages and slow detection
- **Targeting 100% availability**, which means never changing anything
- **Watching averages** while the p99 is catastrophic
- **No external synthetic check**, so DNS and certificate failures look healthy from inside
- **Debugging during the incident** instead of rolling back
- **Reviews that assign blame**, which stops people reporting near-misses
- **Reviews with no owned actions**, so the same incident recurs
- **High-cardinality metric labels**, which destroy the metrics backend
- **Secrets and personal data in logs**
- **Monitoring that depends on the monitored system**
- **On-call without the access needed to fix anything**

---

## 20. Open Source Technologies

- **Prometheus** + **Alertmanager** — metrics and the best open-source alert routing, grouping and silencing
- **Grafana** — dashboards, and SLO/burn-rate panels
- **OpenTelemetry** — instrument once for metrics, logs and traces
- **Loki**, **OpenSearch** — log aggregation and query
- **Jaeger**, **Tempo** — distributed tracing
- **Blackbox exporter**, **k6**, **Checkly-style probes** — synthetic checks from outside
- **Pyrra**, **sloth** — generate SLO and burn-rate alert rules from a definition
- **Vector**, **Fluent Bit** — redact and shape telemetry before it is stored
- **Cachet** and similar — a status page hosted away from your own infrastructure

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Write one SLI and one SLO for your most important user journey.
- [ ] Calculate the error budget that implies, and check last month against it.
- [ ] List every alert and mark those that have ever led to an action. Delete the rest.
- [ ] Add a runbook link to your noisiest alert.
- [ ] Add one synthetic check from outside your network.
- [ ] Count the pages in your last on-call rotation. If it was more than two, that is the problem to fix.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
SLI (user-facing measurement) → SLO → error budget → burn-rate alert
                             → on-call + runbook → mitigate → blameless review
```

## 2. Request Flow

```text
Input       measurements of what users actually experience
    ↓
Processing  compared against an agreed objective; budget burn rate evaluated
    ↓
Output      a small number of pages that are always worth acting on
```

## 3. Real-World Usage

The practices that spread from Google's SRE work — SLOs, error budgets, blameless reviews, symptom-based alerting — became standard because they solve an organisational problem rather than a technical one: they make reliability negotiable with numbers instead of opinions. Mature teams look unimpressive from outside: a handful of alerts, a runbook each, and a habit of deleting the ones that stop earning their place.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Measuring what "working" means, and paging only when it is not |
| **Why does it exist?** | Because users should not be your detection system, and noise is worse than silence |
| **Where does it belong?** | Around everything, feeding a deliberately small set of alerts |
| **When should I use it?** | Before production; and prune it continuously afterwards |
