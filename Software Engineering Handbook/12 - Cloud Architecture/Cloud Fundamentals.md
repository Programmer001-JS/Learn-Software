# Cloud Fundamentals

> **In one line —** renting someone else's computers by the second, which converts capital expenditure into an operating expense and turns hardware problems into API calls.

| | |
|---|---|
| **Category** | Overview note *(hub for this sub-section)* |
| **Architectural Layer** | Infrastructure |
| **Sub-topics** | [IaaS PaaS SaaS](IaaS%20PaaS%20SaaS.md) · [AWS Architecture](AWS%20Architecture.md) · [Cloud Security](Cloud%20Security.md) · [Cloud Monitoring](Cloud%20Monitoring.md) |
| **Related notes** | [Serverless](../10%20-%20Distributed%20Systems/Serverless.md) · [Horizontal Scaling](../14%20-%20Scalability%20and%20Reliability/Horizontal%20Scaling.md) · [High Availability](../14%20-%20Scalability%20and%20Reliability/High%20Availability.md) · [Terraform](../13%20-%20DevOps%20and%20Delivery/Terraform.md) |

---

## 1. Short Definition

*What is it?*

Cloud computing is **on-demand access to computing resources over an API**, billed by usage rather than owned outright. You ask for a server, a database or a terabyte of storage; it exists seconds later; you stop paying when you delete it.

The word "cloud" hides the important part. The important part is not that the machines are remote — it is that **infrastructure became programmable**.

---

## 2. Problem

*What engineering problem does it solve?*

```text
Traditional infrastructure
    ↓
Estimate peak load 18 months from now
    ↓
Buy servers for that peak
    ↓
Wait 6-12 weeks for delivery and racking
    ↓
Run at 15% utilisation forever
    ↓
Get the estimate wrong in either direction and you are stuck
```

Two failures were built into that model: **you paid for capacity you never used**, and **you could not get capacity you suddenly needed**. Both come from the same root cause — the purchase decision happened long before the demand was known.

> [!IMPORTANT]
> **The cloud's real innovation is the feedback loop, not the price.** Provisioning went from months to seconds, which means a wrong capacity decision now costs minutes to correct instead of a purchase order. Everything else — elasticity, managed services, global regions — follows from that one change.

---

## 3. Architecture Position

```text
YOUR APPLICATION
    ↓ uses
MANAGED SERVICES        databases, queues, storage, functions
    ↓ run on
VIRTUALISED COMPUTE     instances, containers, isolated networks
    ↓ run on
PROVIDER HARDWARE       data centres, servers, switches, power, cooling
    ↑
    │ you never see below this line
```

Everything the cloud offers is a decision about **where you draw the line**. Higher up means less control and less work. Lower down means more control and more work. There is no correct height — only a correct height for your team and your problem.

---

## 4. The five properties that define it

```text
ON-DEMAND SELF-SERVICE   an API call, no human in the loop
BROAD NETWORK ACCESS     reachable over standard protocols
RESOURCE POOLING         your instance shares physical hardware
RAPID ELASTICITY         scale up and down within minutes
MEASURED SERVICE         metered billing per second, per GB, per request
```

> [!TIP]
> **Resource pooling is the one people forget, and it explains most mystery performance problems.** Your virtual machine sits on a physical host beside other tenants. "Noisy neighbour" contention on disk and network is real, it is invisible from inside your instance, and it is why two identical instances sometimes perform differently.

---

## 5. Regions, availability zones, edge

```text
REGION            a geographic area — eu-central-1, us-east-1
    │             separate power, separate failure domain, own data residency
    ├── AZ-a      one or more data centres
    ├── AZ-b      independent power and cooling, single-digit ms apart
    └── AZ-c
EDGE LOCATIONS    hundreds of small sites for caching and TLS termination
```

```text
Multi-AZ     → survives a data centre failure    → cheap, do it
Multi-region → survives a region failure         → expensive, rarely needed
```

> [!CAUTION]
> **Multi-AZ and multi-region are not two points on one scale — they are different engineering problems.** Multi-AZ is close to a configuration flag, because latency between zones is a few milliseconds and one synchronous system can span them. Multi-region means tens or hundreds of milliseconds, which forces asynchronous replication, conflict resolution and split-brain handling into your application design. A team that says "we'll go multi-region later" is describing a rewrite.

---

## 6. The deployment models

| Model | What it means | When it is the right answer |
|---|---|---|
| **Public cloud** | Shared provider infrastructure | Almost always, for new systems |
| **Private cloud** | Cloud-style APIs on your own hardware | Regulatory or physical constraints |
| **Hybrid** | Both, connected | An existing data centre you cannot leave yet |
| **Multi-cloud** | Deliberately on two providers | Far rarer than the amount of discussion suggests |

> [!CAUTION]
> **Multi-cloud is usually a cost with no matching benefit.** Two providers means you can only use the lowest common denominator of both, you pay egress to move data between them, and you need people fluent in two operational models. The honest reasons to do it are a contractual requirement or an acquisition. "Avoiding lock-in" is not one, because the portability layer you build to stay neutral is itself a lock-in — one that nobody outside your company maintains.

---

## 7. What you actually pay for

```text
COMPUTE      per second of instance time, or per request
STORAGE      per GB-month, plus a charge per operation
NETWORK      inbound usually free, OUTBOUND ALWAYS COSTS
REQUESTS     per API call on managed services
IDLE         a stopped instance still bills for its attached disk
```

> [!IMPORTANT]
> **Egress is the line item that surprises everyone.** Data leaving the provider is charged per gigabyte at a rate that is high relative to storage. A system serving media, exporting large reports or synchronising data to another cloud can spend more on bandwidth than on servers. Keep bytes inside one region where you can, and put a CDN in front of anything large and repeatedly requested.

---

## 8. The shared responsibility model

```text
PROVIDER SECURES         "security OF the cloud"
    physical data centres, hypervisor, managed service internals
─────────────────────────────────────────────────────────────
YOU SECURE               "security IN the cloud"
    identity and permissions, network rules, encryption choices,
    patching your instances, your application code, your data
```

> [!CAUTION]
> **Nearly every headline cloud breach sits on your side of that line.** The public storage bucket, the over-permissive role, the database open to the internet — the provider's infrastructure worked exactly as designed. Moving to the cloud removes some categories of risk and adds one large one: **a single leaked credential can now reach everything, from anywhere.** See [Cloud Security](Cloud%20Security.md) and [IAM](IAM.md).

---

## 9. Real World Example

- **Netflix** — the reference case for building at scale on public cloud; see [Netflix Architecture](../15%20-%20Real%20World%20System%20Design/Netflix%20Architecture.md).
- **Any startup since roughly 2010** — no capital outlay before the first customer.
- **Seasonal retail** — capacity for Black Friday that is released in December.
- **Batch and ML training** — thousands of cores for one hour, which nobody would buy outright.
- **Disaster recovery** — a standby environment costing almost nothing until needed; see [Disaster Recovery](../14%20-%20Scalability%20and%20Reliability/Disaster%20Recovery.md).

---

## 10. Communication and Dependencies

- **An identity system** — every action is an authenticated API call; see [IAM](IAM.md)
- **A network boundary** you define yourself; see [VPC](VPC.md)
- **Infrastructure as code**, or your environment becomes undocumented; see [Terraform](../13%20-%20DevOps%20and%20Delivery/Terraform.md)
- **Monitoring**, because you cannot walk over to the machine; see [Cloud Monitoring](Cloud%20Monitoring.md)
- **A billing alarm** — the dependency people add only after the first bad invoice

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Use the cloud when your load is uncertain or variable, when you are small enough that operating hardware is a distraction, or when you need a service — object storage, a global CDN, a managed database — that you would never build well yourself.

> [!CAUTION]
> - **Not when load is large, flat and predictable** — at steady high utilisation, owned hardware is genuinely cheaper
> - **Not to fix an architecture problem** — a slow monolith is a slow monolith on rented hardware too
> - **Not without cost visibility** — elasticity with no alarms is an unbounded liability
> - **Not for data you are legally forbidden to place there** — check residency before designing
> - **Not lifted-and-shifted and then declared finished** — that buys the cost of the cloud with the rigidity of a data centre

---

## 12. Advantages and Disadvantages

**Advantages**
- Capacity in seconds, released in seconds
- No capital expenditure and no hardware lifecycle
- Managed services replace entire operational disciplines
- Multi-AZ and multi-region redundancy as configuration
- Global reach without global offices
- A large, transferable ecosystem of tools and knowledge

**Disadvantages**
- **More expensive at steady high utilisation** than owned hardware
- Costs are variable and easy to lose control of
- Real lock-in through managed services and data gravity
- Less visibility below your layer — you debug with the provider's metrics
- Shared tenancy introduces performance variability
- Egress charges shape architecture in ways that feel arbitrary
- The provider's outage is your outage, and you can only wait

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Provisioning** | Seconds to minutes, versus weeks |
| **Network within an AZ** | Sub-millisecond; treat as local |
| **Network across AZs** | Single-digit milliseconds; fine for synchronous replication |
| **Network across regions** | Tens to hundreds of milliseconds; forces asynchronous design |
| **Instance variability** | Shared hardware means identical instances can differ |
| **Managed service limits** | Quotas and throttles that fail differently from hardware |

> [!TIP]
> **Learn the service quotas before you load test, not during the incident.** Cloud services do not degrade gracefully under load the way a server does — they return a throttling error at a hard limit. Your retry and backoff behaviour is therefore part of your performance characteristics, not an afterthought.

---

## 14. Security Considerations

> [!CAUTION]
> **The cloud's control plane is an internet-facing API that can create, read and destroy your entire infrastructure.** An attacker with sufficient credentials does not need to breach a network perimeter — there is no perimeter to breach. This is the single most important mental shift: **identity is the perimeter.**

- **Least privilege on every role**, and no long-lived access keys; see [IAM](IAM.md)
- **Multi-factor authentication on the root account**, then lock it away
- **Private subnets by default** — nothing gets a public address without a reason; see [VPC](VPC.md)
- **Encryption at rest and in transit** — effectively free, so there is no argument against it
- **Audit logging enabled everywhere and shipped to a separate account**, so an intruder cannot erase the trail
- **Public storage is opt-in** — treat every opt-in as an incident to review

---

## 15. Mental Model

> [!NOTE]
> **The cloud is electricity rather than a generator.**
>
> Nobody buys a generator to light an office. You connect to the grid, pay for what you draw, and someone else worries about turbines. You give up the ability to choose your own voltage, you are affected when the grid fails, and at very large and constant consumption it becomes cheaper to generate your own. Every cloud trade-off is in that analogy.

---

## 16. Mini Architecture Diagram

```text
                        INTERNET
                            │
                    ┌───────▼────────┐
                    │  Edge / CDN    │  cached and terminated near users
                    └───────┬────────┘
                            │
    ┌───────────────── REGION ───────────────────┐
    │                                             │
    │   ┌──────── AZ-a ────────┐  ┌── AZ-b ───┐  │
    │   │  Load balancer node  │  │  node     │  │
    │   │  App instances       │  │  App      │  │
    │   │  DB primary          │  │  standby  │  │
    │   └──────────────────────┘  └───────────┘  │
    │                                             │
    │   Object storage · Queues · Functions       │
    │   (regional, already redundant)              │
    └─────────────────────────────────────────────┘
              │
        Control plane API  ← IAM authorises every single call
```

---

## 17. Complete Request Flow

```text
Engineer commits infrastructure code
    ↓
CI applies it through the provider API
    ↓
IAM authorises each call → resources exist within seconds
    ↓
─────────────── runtime ───────────────
User request reaches the nearest edge location
    ↓
TLS terminated there, cache checked
    ↓
Forwarded to the load balancer in the region
    ↓
Routed to a healthy instance in whichever AZ has one
    ↓
Instance reads from the managed database (primary in AZ-a)
    ↓
Response returns — egress metered on the way out
    ↓
─────────────── failure ───────────────
AZ-a loses power
    ↓
Health checks fail; the load balancer stops routing there
    ↓
The managed database promotes the AZ-b standby automatically
    ↓
Service continues — degraded in capacity, not in availability
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> The cloud's value is that infrastructure became programmable and capacity became reversible — but it moves security to identity, makes cost a design constraint, and remains genuinely more expensive than hardware for steady, predictable load.

---

## 19. Common Mistakes

- **Lift and shift, then stop** — cloud prices for data centre architecture
- **No cost alarms** until the first shocking invoice
- **Treating egress as free** and designing chatty cross-region data flows
- **Ignoring the shared responsibility model** and assuming the provider secures your data
- **Long-lived access keys** in code, CI or a developer's laptop
- **Everything in one AZ**, then surprise at a data centre outage
- **Clicking resources into existence** in a console, leaving no record of them
- **Choosing multi-cloud for lock-in reasons** and inheriting the worst of both
- **Not knowing service quotas** until throttling appears in production

---

## 20. Open Source Technologies

- **Terraform / OpenTofu**, **Pulumi** — provision cloud resources as code
- **Kubernetes** — the closest thing to a portable compute abstraction
- **OpenStack** — private cloud, if you genuinely must
- **LocalStack** — emulate AWS services locally for tests
- **Prometheus**, **Grafana**, **OpenTelemetry** — monitoring you own rather than rent
- **Infracost** — shows the cost of an infrastructure change inside the pull request

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Open your provider's bill and rank the line items. Explain the top three out loud.
- [ ] Find one resource in your account that no infrastructure code created.
- [ ] Set a billing alarm at 150% of a normal month, if one does not exist.
- [ ] Write down which of your systems survive an AZ failure — and which do not.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Provider hardware → virtualised compute → managed services → your application
                       (you choose where to draw the line)
```

## 2. Request Flow

```text
Input       an authenticated API call, or a user request at the edge
    ↓
Processing  provider capacity allocated on demand, metered, spread across AZs
    ↓
Output      infrastructure or a response — billed per second, per GB, per request
```

## 3. Real-World Usage

Essentially every system built in the last fifteen years starts here, and the lesson that has held up is not "cloud is cheaper" — it is that **reversible capacity decisions beat correct ones**. The teams that struggle are those that moved their servers without moving their assumptions.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | On-demand, metered computing resources behind an API |
| **Why does it exist?** | Because buying capacity ahead of demand always guesses wrong |
| **Where does it belong?** | Underneath everything — it is the substrate, not a component |
| **When should I use it?** | Variable or uncertain load, small teams, or services you would never build well yourself |
