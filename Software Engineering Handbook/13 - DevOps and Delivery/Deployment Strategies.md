# Deployment Strategies

> **In one line —** how new code reaches production without an outage; and the strategy matters far less than whether you can detect a problem and reverse it in minutes.

| | |
|---|---|
| **Category** | Concept / Practice |
| **Architectural Layer** | Delivery |
| **Related notes** | [CI CD](CI%20CD.md) · [Kubernetes](Kubernetes.md) · [Load Balancing](../14%20-%20Scalability%20and%20Reliability/Load%20Balancing.md) · [Monitoring and Alerting](../14%20-%20Scalability%20and%20Reliability/Monitoring%20and%20Alerting.md) · [High Availability](../14%20-%20Scalability%20and%20Reliability/High%20Availability.md) · [Database Fundamentals](../08%20-%20Databases%20and%20Data/Database%20Fundamentals.md) |

---

## 1. Short Definition

*What is it?*

A deployment strategy is the **procedure for replacing running version N with version N+1**: how traffic moves, how failure is detected, and how the change is reversed. It is a risk management decision, not a technical preference.

---

## 2. Problem

*What engineering problem does it solve?*

```text
Version N is serving users. Version N+1 must replace it.
    ↓
Stop everything and start the new one?      → downtime
Replace all instances at once?              → a bug affects 100% instantly
Trust that tests caught everything?         → they did not
    ↓
And the real question nobody asks until it is too late:
    how do we get BACK to version N, and how fast?
```

> [!IMPORTANT]
> **Every strategy here is a way of limiting how many users see a bad release, and for how long.** The metrics that matter are therefore not deployment frequency but **time to detect** and **time to revert**. A team with a plain rolling deployment, good alarms and a one-command rollback is in a far better position than a team with an elaborate canary pipeline and no idea what "healthy" looks like.

---

## 3. Architecture Position

```text
   artefact (image tagged with a commit SHA)
        │
   ┌────▼──── the strategy decides HOW traffic shifts ────┐
   │                                                       │
   │   load balancer / service mesh / feature flag         │
   │        │                                               │
   │   ┌────▼─────┐        ┌──────────┐                    │
   │   │ version N│        │ version  │                    │
   │   │ (old)    │        │ N+1 (new)│                    │
   │   └──────────┘        └──────────┘                    │
   └───────────────────────┬───────────────────────────────┘
                           │
              monitoring decides: proceed or revert
```

---

## 4. The strategies

```text
RECREATE                  stop all old, start all new
    ▪ downtime, simple, sometimes acceptable at 3 a.m. internally

ROLLING                   replace instances a few at a time
    ▪ no downtime, N and N+1 coexist, gradual
    ▪ the default nearly everywhere — and good enough for most teams

BLUE / GREEN              two full environments; switch traffic at once
    ▪ instant rollback, instant cutover
    ▪ double the infrastructure during the switch

CANARY                    send a small % of real traffic to N+1, then grow
    ▪ smallest blast radius; needs good metrics to judge

A/B TESTING               route by user attribute to compare BEHAVIOUR
    ▪ a product experiment, not a deployment safety mechanism

SHADOW / MIRROR           duplicate real traffic to N+1, discard responses
    ▪ tests real load safely; ⚠ side effects must be suppressed
```

> [!TIP]
> **Start with rolling, add canary when you have metrics good enough to judge one, and use blue/green when you need an instant, complete rollback.** These are not maturity levels to climb — they are answers to different questions. A rolling deployment with automated rollback on error rate covers the large majority of real risk.

---

## 5. Rolling deployments, and the constraint everyone forgets

```text
During a rolling deployment, N and N+1 RUN SIMULTANEOUSLY
    ↓
Which means BOTH must work with:
    the same database schema
    the same message formats on the queue
    the same cache entries
    the same API contract with each other
```

> [!CAUTION]
> **Any rolling or canary strategy requires N and N+1 to be mutually compatible, and this is where most deployment incidents actually come from.** A change that renames a column, alters a serialised cache value or changes a queue message shape will work perfectly in testing and fail during the twenty minutes when both versions are live. Design every change to be backward compatible for one release — it is the price of zero-downtime deployment.

---

## 6. Canary, done properly

```text
5% of traffic to N+1
    ↓
Compare AGAINST N, on the same window:
    error rate · p99 latency · saturation · business metrics
    ↓
Statistically worse?  → automatic rollback
Healthy for 10 min?   → 25% → 50% → 100%
```

```text
A CANARY WITHOUT AUTOMATED ANALYSIS IS JUST A SLOW DEPLOYMENT
    a human watching a dashboard for ten minutes will miss it
    or get distracted, or be asleep
```

> [!IMPORTANT]
> **Compare the canary against the current version, not against a fixed threshold.** Absolute thresholds fail in both directions: traffic patterns vary by hour, so a 1% error rate may be normal at peak and alarming at night. Comparing N+1's error rate to N's over the same window controls for everything you did not think of. This is what Argo Rollouts and Flagger automate, and it is the difference between a canary and theatre.

---

## 7. Blue/green: instant rollback, at a price

```text
BLUE (live, version N)          GREEN (idle, version N+1)
        ▲                                │
        │  load balancer / DNS           │  deploy and smoke test here
        └──── switch ────────────────────┘
                    ↓
        traffic moves in one step; BLUE stays warm
                    ↓
        problem? switch back — seconds, not a redeployment
```

```text
THE COSTS
    double infrastructure during the overlap
    the database is usually SHARED → schema must serve both
    long-lived connections and in-flight requests need draining
    DNS-based switching is slow and unreliable (TTLs, client caching)
```

> [!TIP]
> **Switch at the load balancer, not with DNS.** DNS TTLs are advisory, clients cache aggressively, and some libraries never re-resolve — so a "instant" DNS cutover can leave traffic on the old environment for hours, and rollback is equally slow. Target groups or a service mesh give you an actual switch.

---

## 8. Database migrations: the part that cannot roll back

```text
Code rolls back in seconds.  A dropped column does not.
```

```text
EXPAND / MIGRATE / CONTRACT — across THREE releases
    1. EXPAND    add the new column, write to BOTH, read the old one
                 (fully backward compatible; safe to roll back)
    2. MIGRATE   backfill, then switch reads to the new column
                 (still safe: the old column is intact)
    3. CONTRACT  stop writing the old column, then drop it
                 (days later, once rollback is no longer plausible)
```

> [!CAUTION]
> **A destructive migration in the same release as the code that needs it makes rollback impossible, and it will be discovered during an incident.** Additive changes are always safe; removals must lag behind by at least one release. Also beware migrations that lock a large table — an `ALTER` that takes a lock for four minutes on a busy table is an outage regardless of your deployment strategy. See [Database Fundamentals](../08%20-%20Databases%20and%20Data/Database%20Fundamentals.md).

---

## 9. Feature flags: separating deploy from release

```text
DEPLOY   the code is in production      (technical, low risk, frequent)
RELEASE  users experience the change    (business decision, reversible instantly)
```

```text
WITH FLAGS
    merge incomplete work safely
    enable for internal users → 1% → 50% → everyone
    disable in SECONDS without a deployment
    ↓
The fastest rollback available is a flag, not a redeployment
```

> [!TIP]
> **A flag is the only rollback that takes effect in seconds and needs no pipeline.** For risky behavioural changes, ship the code dark and turn it on separately — then a problem is a configuration change rather than an emergency release. The discipline is removing flags afterwards: a two-year-old flag is an untested hidden code path.

---

## 10. Real World Example

- **Kubernetes rolling updates** — the default, gated by readiness probes; see [Kubernetes](Kubernetes.md).
- **Progressive canary with automated analysis** — Argo Rollouts or Flagger comparing metrics against the stable version.
- **Blue/green on AWS** — two target groups behind one ALB, switched in one action.
- **Mobile applications** — staged store rollouts at 1%, 10%, 50%, since rollback means shipping a new build.
- **Netflix and similar** — canary analysis with automated judgement at large scale; see [Netflix Architecture](../15%20-%20Real%20World%20System%20Design/Netflix%20Architecture.md).
- **Shadow traffic** before a major rewrite goes live, to test real load without user impact.

---

## 11. Communication and Dependencies

- **A health check that reflects readiness**, not just process liveness
- **Monitoring good enough to judge a release**; see [Monitoring and Alerting](../14%20-%20Scalability%20and%20Reliability/Monitoring%20and%20Alerting.md)
- **A load balancer or mesh** that can shift traffic by weight
- **Immutable, tagged artefacts**, so the previous version still exists
- **Backward-compatible schema changes** as a standing rule
- **Connection draining**, or in-flight requests are dropped at every switch
- **A feature flag system**, for the fastest reversal path

---

## 12. When To Use / When NOT To Use

> [!TIP]
> Rolling deployment with health-gated replacement and automated rollback covers most systems. Add canary analysis where a bad release is expensive, and blue/green where you need to revert an entire environment at once.

> [!CAUTION]
> - **Do not attempt canary without metrics you trust** — you are deploying slowly and calling it safety
> - **Do not use blue/green** if your database schema cannot serve both versions
> - **Do not use A/B testing as a safety mechanism** — it answers a product question, not an operational one
> - **Do not use shadow traffic** without disabling writes, emails, payments and third-party calls
> - **Do not deploy zero-downtime** while shipping a destructive migration
> - **Do not choose an elaborate strategy** to compensate for the absence of monitoring

---

## 13. Advantages and Disadvantages

**Advantages of progressive strategies**
- Blast radius limited to a small fraction of users
- Real production traffic reveals what staging cannot
- Rollback becomes routine rather than an incident
- Deployments can happen during the day, which is when people are awake to fix them
- Confidence rises, so releases get smaller and more frequent

**Disadvantages**
- **N and N+1 must be compatible**, which constrains every change
- Extra infrastructure during overlap
- Automated analysis needs reliable metrics and enough traffic to be significant
- Long-lived connections complicate every traffic shift
- Feature flags accumulate into hidden complexity
- More moving parts in the delivery pipeline itself

---

## 14. Performance Impact

| Aspect | Impact |
|---|---|
| **Rolling** | Reduced capacity during the roll — keep surge headroom |
| **Blue/green** | Double infrastructure temporarily; a cold new environment may be slow at first |
| **Canary** | Small footprint; a low-traffic canary needs longer to be statistically meaningful |
| **Connection draining** | Adds time per instance; too short drops requests |
| **Cold caches** | New instances start empty — a mass replacement can cause a latency spike |
| **Shadow** | Doubles downstream load; the new version's dependencies must cope |

> [!TIP]
> **Cold caches are the most common cause of a "successful" deployment causing a latency spike.** Replacing all instances at once empties every in-process cache and hits the database with the full load. Roll gradually, warm caches during startup before the readiness probe passes, or both.

---

## 15. Security Considerations

> [!CAUTION]
> **A deployment pipeline that can change production is a privileged path, and rollback must not be a way around your controls.** An emergency rollback that skips image signature verification, or a break-glass deployment that bypasses review, is exactly the mechanism an attacker wants. Make the fast path *pre-approved* — a previously deployed, already verified artefact — rather than unverified.

- **Roll back to a previously verified artefact**, never to an unsigned or hand-built one
- **Secrets rotate separately from deployment**, so a rollback does not restore an old credential
- **Both versions must be authorised** — if N+1 needs new permissions, granting them early is safer than granting them mid-rollout
- **Shadow traffic must not reach real third parties**, or you will double-charge customers
- **Audit who deployed what, and when**; environment protection with required reviewers for production
- **Feature flags are production configuration** — access to the flag system is access to behaviour

---

## 16. Mental Model

> [!NOTE]
> **Deployment strategies are how you cross a river.**
>
> Recreate is jumping. Rolling is stepping across stones one at a time — you are briefly on both banks, which is exactly why both must hold your weight. Blue/green is building a second bridge and switching everyone over at once, with the old one still standing. Canary is sending one person over first and watching. Every one of them depends on being able to turn round — and the only thing that truly makes crossing safe is knowing quickly that the stone is loose.

---

## 17. Mini Architecture Diagram

```text
                        load balancer / mesh
                                │
        ┌───────────────────────┼───────────────────────┐
        │                       │                       │
   95% ▼                    5% ▼                        │
  ┌──────────┐            ┌──────────┐                  │
  │ STABLE   │            │  CANARY  │                  │
  │ version N│            │version N+1│                 │
  │ 10 pods  │            │  1 pod   │                  │
  └──────────┘            └────┬─────┘                  │
        │                      │                        │
        └──────────┬───────────┘                        │
                   ▼                                    │
        ┌──────────────────────────┐                    │
        │  METRIC COMPARISON        │                    │
        │  error rate: N vs N+1     │                    │
        │  p99 latency: N vs N+1    │                    │
        │  over the SAME window     │                    │
        └────────┬─────────────────┘                    │
        worse ───┘         └─── healthy                  │
          │                        │                     │
   AUTOMATIC ROLLBACK      25% → 50% → 100%              │
                                                         │
   SHARED DATABASE ◄─── schema must serve BOTH ──────────┘
   (expand / migrate / contract, across three releases)

   FEATURE FLAG ── the fastest reversal: seconds, no deployment
```

---

## 18. Complete Request Flow

```text
CI produces image app:<sha>; the previous image still exists
    ↓
Canary rollout starts: one new pod, 5% of traffic
    ↓
Readiness probe passes only after the cache is warmed
    ↓
For 10 minutes, error rate and p99 for N+1 are compared with N
    ↓
Both within tolerance → traffic shifts to 25%, then 50%, then 100%
    ↓
Old pods drained: they stop accepting new connections, finish in-flight requests, exit
    ↓
The new feature is deployed but OFF behind a flag
    ↓
Product enables it for internal users, then 1%, then everyone — no deployment
    ↓
─────────────── a bad release ───────────────
At 5%, error rate for N+1 is 6× the stable version
    ↓
Automated analysis fails the canary
    ↓
Traffic returns to N; the canary pod is removed — 90 seconds total
    ↓
Impact: 5% of traffic for 3 minutes. Nobody was paged.
    ↓
─────────────── a bad release that tests missed ───────────────
The bug is behavioural, not an error: a pricing calculation is wrong
    ↓
Error rate looks perfect; the business metric on the canary does not
    ↓
Which is why canary analysis must include business metrics, not just health
    ↓
─────────────── the migration ───────────────
Release 1: add `email_verified_at`, write both columns, read the old
Release 2: backfill, switch reads to the new column
Release 3 (a week later): stop writing the old column
Release 4 (later still): drop it
    ↓
At every point, rolling back one release is safe
```

---

## 19. Key Takeaway

> [!IMPORTANT]
> Every strategy is about limiting blast radius and reversing quickly — make N and N+1 compatible, never ship a destructive migration with the code that needs it, automate the rollback decision from metrics, and remember a feature flag reverts faster than any deployment.

---

## 20. Common Mistakes

- **A destructive migration in the same release** as the code, making rollback impossible
- **Assuming N and N+1 never coexist** — in any rolling deployment they do
- **A canary with no automated analysis** — a slow deployment wearing a safety label
- **Absolute thresholds instead of comparison** against the stable version
- **DNS-based blue/green switching**, so cutover and rollback both take hours
- **No connection draining**, dropping in-flight requests on every deployment
- **Replacing all instances at once**, then a latency spike from cold caches
- **A health check that only proves the process started**
- **Shadow traffic with live side effects** — duplicate emails, duplicate charges
- **A/B testing mistaken for a safety mechanism**
- **Feature flags never removed**, becoming permanent untested branches
- **A rollback path that has never been used** before the day it is needed
- **Deploying only at night**, when the people who could fix it are asleep

---

## 21. Open Source Technologies

- **Argo Rollouts**, **Flagger** — canary and blue/green with automated metric analysis on Kubernetes
- **Argo CD**, **Flux** — GitOps delivery, where rollback is reverting a commit
- **Istio**, **Linkerd**, **Envoy** — traffic weighting and mirroring at the mesh layer
- **Spinnaker** — multi-cloud progressive delivery with judgement stages
- **Unleash**, **Flagsmith**, **OpenFeature** — self-hosted feature flags
- **Kubernetes Deployments** — rolling updates and `rollout undo` out of the box
- **gh-ost**, **pt-online-schema-change** — non-blocking schema changes on MySQL
- **k6**, **Vegeta** — verify a new version under load before promoting it

---

## 22. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 23. Workbook Exercise

- [ ] Time a rollback in staging. That number is your worst-case exposure.
- [ ] Review your last schema migration: could the previous release have run against it?
- [ ] Check whether your health check would fail if the database were unreachable — and decide whether it should.
- [ ] Confirm connection draining is configured, and long enough for your slowest request.
- [ ] List your feature flags and delete one that is permanently on.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
artefact → traffic shifted gradually (rolling / canary / blue-green)
                → metrics compared against stable → promote or revert
                → feature flag controls user-visible release separately
```

## 2. Request Flow

```text
Input       a new immutable artefact, and a healthy running version
    ↓
Processing  traffic shifted incrementally while N and N+1 coexist compatibly
    ↓
Output      a promoted release, or an automatic reversal before most users noticed
```

## 3. Real-World Usage

The organisations that deploy most often are also the ones with the lowest failure rate, and the mechanism is visible in their deployment strategies: small changes, progressive traffic shifts, automated judgement from metrics, and rollback treated as routine. Nothing in that list is exotic — the hard part is the standing discipline of backward-compatible changes.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | The procedure for replacing a running version without an outage |
| **Why does it exist?** | Because tests do not catch everything, and users should not be the detector |
| **Where does it belong?** | Between your artefact and your traffic |
| **When should I use it?** | Always — and pick the simplest one your monitoring can actually judge |
