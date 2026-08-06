# Disaster Recovery

> **In one line —** the plan for getting the whole system back after something large goes wrong; and its quality is measured entirely by the last time you rehearsed it.

| | |
|---|---|
| **Category** | Concept / Practice |
| **Architectural Layer** | Cross-cutting |
| **Related notes** | [Backup Strategy](Backup%20Strategy.md) · [High Availability](High%20Availability.md) · [Terraform](../13%20-%20DevOps%20and%20Delivery/Terraform.md) · [Cloud Fundamentals](../12%20-%20Cloud%20Architecture/Cloud%20Fundamentals.md) · [Monitoring and Alerting](Monitoring%20and%20Alerting.md) |

---

## 1. Short Definition

*What is it?*

Disaster recovery is the set of **plans, capabilities and rehearsals** for restoring service after an event too large for normal failover: a lost region, a destroyed account, ransomware, or a mistake that deleted production.

---

## 2. Problem

*What engineering problem does it solve?*

```text
High availability handles a failed instance, a failed AZ, a failed database node
    ↓
It does NOT handle
    an entire region becoming unavailable
    an account deleted or compromised
    ransomware across every environment
    `terraform destroy` in the wrong directory
    a provider suspending your account over a billing dispute
    ↓
These are not component failures. They remove the whole environment.
```

> [!IMPORTANT]
> **High availability and disaster recovery are different disciplines answering different questions.** HA asks "how do we not go down when a part fails?" DR asks "how do we come back when everything is gone?" The first is architecture; the second is a documented, rehearsed *procedure* with named owners. A system can have perfect HA and no DR — and many do.

---

## 3. Architecture Position

```text
   NORMAL FAILURE            → handled by HA, automatically, no human
   ────────────────────────────────────────────────────────────────
   DISASTER                  → handled by DR, deliberately, by people
        │
   ┌────┴──────────────────────────────────────────┐
   │  what you need to have READY BEFOREHAND:      │
   │                                                │
   │  infrastructure as code    (rebuild the env)   │
   │  backups in another account (get the data)     │
   │  DNS control               (redirect traffic)  │
   │  a written runbook         (know the order)    │
   │  credentials reachable     (be able to log in) │
   │  a rehearsal               (know it works)     │
   └───────────────────────────────────────────────┘
```

---

## 4. RPO and RTO, again — because they drive the cost

```text
RPO  how much data may be lost      → replication frequency
RTO  how long recovery may take      → the strategy you must pay for
```

```text
STRATEGY            RTO            RPO          COST
backup & restore    hours-days     hours        LOWEST
pilot light         tens of min    minutes      low
warm standby        minutes        seconds      medium
active-active       ~zero          ~zero        HIGHEST
```

> [!TIP]
> **Choose the cheapest strategy that meets a number the business has actually agreed to.** Almost every team overestimates what they need in a workshop and underinvests in practice. A rehearsed backup-and-restore with a four-hour RTO is worth far more than an unrehearsed multi-region active-active setup — because the second one has never been tried and will not work the first time.

---

## 5. The four strategies

```text
BACKUP AND RESTORE
    backups in another region/account; nothing else running
    disaster → provision from code, restore data, repoint DNS
    ✓ cheap  ✗ hours, and every step must work under pressure

PILOT LIGHT
    the data layer replicating continuously; compute defined but OFF
    disaster → scale compute up, promote the database, repoint DNS
    ✓ good balance  ✗ needs capacity to be available on the day

WARM STANDBY
    a small but COMPLETE running copy in another region
    disaster → scale it up, shift traffic
    ✓ minutes, and the path is exercised by simply existing

ACTIVE-ACTIVE
    both regions serve traffic all the time
    disaster → remove one from rotation
    ✓ near-zero RTO  ✗ expensive, and forces distributed data design
```

> [!CAUTION]
> **Active-active is not "warm standby but better" — it is a different data architecture.** Serving writes in two regions means either partitioning users by region, accepting eventual consistency with conflict resolution, or adopting a globally distributed database. Teams that budget for active-active as an infrastructure line item and discover it as an application redesign lose a lot of time.

---

## 6. The real single points of failure in a disaster

```text
THINGS PEOPLE FORGET UNTIL THE DAY
    DNS               who can change it, and is the registrar reachable?
    TLS certificates  can you issue new ones in the recovery region?
    secrets           are they in the account you just lost?
    the runbook       is it in the wiki that is also down?
    credentials       is your identity provider a dependency of the recovery?
    third parties     payment, email, SMS — do they allow a new IP range?
    the people        who has the permissions, and are they on holiday?
    quotas            can the recovery region even launch that many instances?
```

> [!CAUTION]
> **Service quotas in the recovery region are a genuinely common blocker.** A fresh region has default limits — instance counts, IP addresses, connections — and raising them requires a support request that takes hours or days. If you have never launched production-sized capacity there, you do not know whether you can. Request the quotas in advance and verify them during the rehearsal.

---

## 7. Infrastructure as code is the precondition

```text
WITHOUT IaC
    "rebuild the environment" means archaeology
    → nobody knows the security group rules, the parameter groups, the DNS records
    → the recovery takes days and produces something subtly different

WITH IaC
    terraform apply in another region
    → 20 minutes to an empty but correct environment
    → then restore data into it
```

> [!IMPORTANT]
> **Disaster recovery is the strongest argument for infrastructure as code, stronger than repeatability or review.** It converts "rebuild everything" from an unbounded research project into a command with a known runtime. If your environment contains resources no code describes, those are the ones that will be missing, and you will discover which during the recovery. See [Terraform](../13%20-%20DevOps%20and%20Delivery/Terraform.md).

---

## 8. The rehearsal is the deliverable

```text
A DR PLAN THAT HAS NEVER BEEN EXECUTED IS A DOCUMENT, NOT A CAPABILITY
```

```text
LEVELS OF REHEARSAL, in increasing value
    1. read the runbook aloud together              (finds missing steps)
    2. tabletop exercise — talk through a scenario   (finds wrong assumptions)
    3. restore data into a scratch environment       (finds broken backups)
    4. full rebuild in another region, timed         (finds everything else)
    5. real failover of production traffic           (the only true proof)
```

> [!TIP]
> **Run the rehearsal with the person who wrote the runbook absent.** The plan needs to work for whoever is on call, not for its author — and the gap between "obvious to me" and "written down" is exactly what a real incident exposes. Time it, record what went wrong, and fix the runbook. The output of a rehearsal is an updated document and a realistic RTO.

---

## 9. Real World Example

- **Cross-region backup copies with a documented rebuild** — the honest baseline for most companies.
- **Pilot light** — an RDS cross-region read replica plus Terraform-defined compute that is not running.
- **Warm standby** — a scaled-down complete environment, promoted and scaled on demand.
- **Active-active by geography** — users partitioned by region, each region authoritative for its own data.
- **A recovery from ransomware** using immutable backups in a separate account; see [Backup Strategy](Backup%20Strategy.md).
- **A provider region outage** where the correct decision was to wait, because the RTO exceeded the expected outage.

---

## 10. Communication and Dependencies

- **Infrastructure as code**, covering everything
- **Backups in a separate account and region**, immutable and tested
- **DNS you can change quickly**, with a low TTL on the records that matter
- **Secrets replicated** to somewhere the disaster does not reach
- **Quotas pre-approved** in the recovery region
- **A runbook outside the primary environment** — printed, or in a different provider
- **A communication plan** — a status page and customer messaging that do not depend on your own systems
- **Named roles**: who declares a disaster, who executes, who communicates

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Every production system needs at least backup-and-restore with a rehearsed runbook. Escalate to pilot light or warm standby when the business can quantify the cost of a multi-hour outage and it exceeds the cost of the standby.

> [!CAUTION]
> - **Do not build multi-region before multi-AZ is solid** — the common failures are smaller than you think
> - **Do not plan for a disaster while ignoring the likely causes** — deployments and human error cause far more downtime than regions failing
> - **Do not write a plan you will not rehearse** — it is worse than none, because it creates false confidence
> - **Do not put the runbook, the secrets and the credentials inside the thing you are recovering from**
> - **Do not skip the "wait" option** — if a provider outage will be over in an hour and your RTO is four, failing over may be the riskier choice
> - **Do not confuse DR with HA**, and do not let a Multi-AZ diagram stand in for a plan

---

## 12. Advantages and Disadvantages

**Advantages**
- Survivability against events that end companies
- Rehearsals surface hidden dependencies you would otherwise meet during an incident
- Forces infrastructure as code and tested backups, which pay off daily
- A realistic RTO enables honest commitments to customers
- Often required for enterprise contracts and compliance

**Disadvantages**
- **Real, ongoing cost** — standby capacity, cross-region transfer, duplicated services
- Rehearsals take engineering time away from features
- Multi-region introduces its own failure modes and latency constraints
- Data consistency across regions is genuinely hard
- Plans decay: every architecture change can invalidate the runbook
- Easy to over-engineer for a scenario far less likely than a bad deployment

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Cross-region replication** | Async, so lag is your RPO; tens to hundreds of ms of latency |
| **DNS TTL** | Determines how fast traffic can be redirected — set it low in advance |
| **Restore throughput** | Usually the dominant term in RTO for large datasets |
| **Cold standby warm-up** | Empty caches and cold connection pools after failover |
| **Quota limits** | Can make the recovery region slower to scale than the primary |
| **Active-active writes** | Cross-region coordination adds latency to every write |

> [!TIP]
> **Lower the TTL on your critical DNS records *before* you need to change them.** A 24-hour TTL means a redirect takes a day to propagate, which makes DNS-based failover useless. Set it to 60 seconds as standing configuration — the extra query volume is negligible and it turns DNS into a usable switch.

---

## 14. Security Considerations

> [!CAUTION]
> **The most likely disaster for a modern company is not a natural one — it is a compromised credential with administrative access.** That scenario has a specific requirement: recovery assets the attacker cannot reach. A separate account, immutable backups, and credentials that are not stored in the compromised environment. If your DR plan assumes you still control the production account, it does not cover the most probable case.

- **Recovery must not depend on the compromised environment** — separate account, separate credentials
- **Immutable backups** with Object Lock; see [Backup Strategy](Backup%20Strategy.md)
- **Break-glass credentials** stored offline, MFA-protected, and their use alarmed
- **Assume the intruder read everything** — rotate all secrets as part of recovery
- **Recovery must not lower your security posture** — a hastily built environment with open security groups is a second incident
- **Preserve evidence before rebuilding**, if the cause may be malicious
- **Audit logs in a separate account**, so you can reconstruct what happened

---

## 15. Mental Model

> [!NOTE]
> **Disaster recovery is a fire drill, not a fire extinguisher.**
>
> The extinguisher is high availability — it handles the small fire automatically, without evacuating. The drill is for when the building is gone: everyone knows where to go, who counts heads, and where the spare keys are. Its value comes entirely from having been practised. A building with a laminated evacuation plan nobody has walked is exactly as prepared as one with no plan — it just feels safer.

---

## 16. Mini Architecture Diagram

```text
   ┌──────── PRIMARY REGION (eu-central-1) ──────────────┐
   │  multi-AZ app · DB primary + standby · caches        │
   │  full production traffic                             │
   └───────┬──────────────────────┬──────────────────────┘
           │ async replication     │ backup copies
           │                       ▼
           │            ┌─── BACKUP ACCOUNT ────────────┐
           │            │  immutable · Object Lock      │
           │            │  cross-region copies          │
           │            └───────────────────────────────┘
           ▼
   ┌──────── DR REGION (eu-west-1) ── PILOT LIGHT ────────┐
   │  DB cross-region read replica    ← RUNNING            │
   │  Terraform-defined compute       ← DEFINED, OFF       │
   │  quotas pre-approved              ← verified          │
   │  secrets replicated               ← present           │
   └──────────────────────────────────────────────────────┘

   DNS: TTL 60 s, health-check failover ready
   RUNBOOK: stored outside both regions, printed copy held
   BREAK-GLASS credentials: offline, MFA, alarmed on use
   REHEARSAL: quarterly, timed, run by someone who did not write it
```

---

## 17. Complete Request Flow

```text
─────────────── the rehearsal (quarterly, planned) ───────────────
On-call engineer opens the runbook. The author is deliberately absent.
    ↓
Step 1: terraform apply in the DR region → 22 minutes
    ↓
Step 2: promote the cross-region read replica → 4 minutes
    ↓
Step 3: restore object storage from the backup account → 35 minutes
    ↓
Step 4: repoint DNS (TTL 60 s) → 2 minutes
    ↓
Step 5: smoke tests → FAIL. The payment provider rejects the new IP range.
    ↓
FINDING: an allowlist nobody knew about. Added to the runbook and pre-approved.
    ↓
Total: 1 h 40 m. Documented RTO updated from "about an hour" to 2 hours.
    ↓
─────────────── the real event ───────────────
The primary region loses a core service; the provider gives no ETA
    ↓
DECISION POINT: fail over, or wait?
    ↓
Historical outages of this type last ~2 h; our RTO is 2 h; failing over
also risks data loss up to the replication lag
    ↓
Decision: wait 45 minutes, prepare in parallel, then decide
    ↓
At 50 minutes the outage widens → disaster declared by the named owner
    ↓
The runbook is executed. Every step has been done before.
    ↓
Service restored in 1 h 50 m, with 90 seconds of data loss (the replica lag)
    ↓
Customers were informed on a status page hosted outside the affected region
    ↓
─────────────── the version without a rehearsal ───────────────
Same event. The plan exists but has never been run.
    ↓
Terraform fails: a resource was created by hand and is not in code
    ↓
Quotas in the DR region cap instances at 20; a support request takes 6 hours
    ↓
The database restore needs a KMS key that lives in the failed region
    ↓
The runbook is in a wiki hosted in the failed region
    ↓
Recovery takes two days
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> Pick the cheapest strategy that meets an agreed RPO and RTO, keep infrastructure in code and backups in a separate account, store the runbook and credentials outside what you are recovering — then rehearse it, because the rehearsal is the only part that produces a real capability.

---

## 19. Common Mistakes

- **Confusing HA with DR**, and treating a Multi-AZ diagram as a plan
- **A plan that has never been rehearsed**
- **The runbook stored in a system that is also down**
- **Secrets and encryption keys only in the environment being recovered**
- **Resources created by hand**, so the rebuild is incomplete
- **Quotas never verified** in the recovery region
- **DNS TTL of hours**, making the redirect useless
- **No named owner** for declaring a disaster, so nobody decides
- **Assuming you still control the production account** — the likeliest disaster is that you do not
- **Ignoring third-party allowlists**, which reject your recovery region
- **Over-engineering for a region outage** while lacking a rollback for a bad deployment
- **Not considering "wait"** as a legitimate option
- **A plan that decays** because architecture changed and nobody updated it

---

## 20. Open Source Technologies

- **Terraform / OpenTofu** — the precondition; rebuild the environment from code
- **Velero** — back up and restore Kubernetes clusters across regions
- **pgBackRest**, **WAL-G** — PostgreSQL PITR and cross-region archives
- **restic**, **rclone** — move data between providers and regions
- **Chaos Mesh**, **LitmusChaos** — practise failure deliberately
- **external-dns**, **cert-manager** — automate the DNS and certificate steps that block recovery
- **Prometheus** + **Blackbox exporter** — external checks that tell you the region is gone
- **A status page hosted elsewhere** — Cachet, or any provider outside your own infrastructure

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Write down your actual RPO and RTO, and get the business to agree to them.
- [ ] Try to rebuild your environment from code in a different region. Note everything missing.
- [ ] Check whether your runbook, secrets and credentials survive losing the primary account.
- [ ] Look up your service quotas in the recovery region.
- [ ] Lower the TTL on your critical DNS records.
- [ ] Schedule a rehearsal, and make sure the author of the runbook is not the one executing it.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
primary region → async replication + immutable backups in a separate account
              → DR region (pilot light / warm standby) → DNS switch → runbook → rehearsal
```

## 2. Request Flow

```text
Input       an event that removes the whole environment
    ↓
Processing  declare, rebuild from code, restore data, redirect traffic, verify
    ↓
Output      service restored within a time you have actually measured
```

## 3. Real-World Usage

The companies that recover well are rarely the ones with the most elaborate architecture — they are the ones who had run the drill. Rehearsals reliably uncover the same categories of problem: a resource not in code, a quota not raised, a key in the wrong place, and a runbook stored inside the thing that failed. None of those are discoverable on paper.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | The rehearsed capability to restore the whole system after a large loss |
| **Why does it exist?** | Because high availability does not cover losing the environment itself |
| **Where does it belong?** | Outside your primary account and region — including the runbook |
| **When should I use it?** | Every production system needs a plan; the strategy depends on an agreed RTO |
