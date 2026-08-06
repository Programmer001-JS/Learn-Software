# IaaS PaaS SaaS

> **In one line —** three answers to one question: how much of the stack do you want to be responsible for?

| | |
|---|---|
| **Category** | Concept / Service Model |
| **Architectural Layer** | Infrastructure |
| **Related notes** | [Cloud Fundamentals](Cloud%20Fundamentals.md) · [EC2](EC2.md) · [Lambda](Lambda.md) · [Serverless](../10%20-%20Distributed%20Systems/Serverless.md) · [Kubernetes](../13%20-%20DevOps%20and%20Delivery/Kubernetes.md) |

---

## 1. Short Definition

*What is it?*

Three **service models** describing where the boundary sits between what the provider operates and what you operate:

```text
IaaS   Infrastructure as a Service   you get a machine
PaaS   Platform as a Service         you get a place to put code
SaaS   Software as a Service         you get a working product
```

---

## 2. Problem

*What engineering problem does it solve?*

```text
Running a web application yourself means owning:
    hardware → power → OS → patching → runtime → dependencies
    → deployment → scaling → backups → monitoring → the application
    ↓
Twelve responsibilities, and only ONE of them is your product
```

The models exist so you can **hand over the layers that are not your competitive advantage**. Nobody wins customers by patching kernels well.

> [!IMPORTANT]
> **This is a staffing decision disguised as a technical one.** Every layer you keep needs someone who understands it, on call, for as long as the system lives. A three-person team choosing IaaS has quietly signed up to be a systems administration team as well as a product team.

---

## 3. Architecture Position

```text
                    IaaS      PaaS      SaaS
Your data            YOU       YOU     you own it, they hold it
Your application     YOU       YOU      provider
Runtime              YOU     provider   provider
OS / patching        YOU     provider   provider
Virtualisation     provider  provider   provider
Hardware           provider  provider   provider
```

Read the table as a single sliding line. There is nothing sacred about the three names — modern services sit anywhere along it, and the useful question is always *which of these rows am I on call for?*

---

## 4. The models in practice

| | You get | You still do | Example |
|---|---|---|---|
| **IaaS** | A virtual machine, a network, a disk | Everything above the OS | [EC2](EC2.md), [VPC](VPC.md) |
| **CaaS** | A container scheduler | Images, manifests, the cluster's health | [Kubernetes](../13%20-%20DevOps%20and%20Delivery/Kubernetes.md), ECS |
| **PaaS** | "Give us your code" | Write the code, set the config | Heroku, App Runner, Vercel |
| **FaaS** | "Give us a function" | Write the function, accept the model | [Lambda](Lambda.md) |
| **SaaS** | A finished product | Configure and use it | Gmail, Stripe, Datadog |

> [!TIP]
> **The two extra letters matter more than the original three.** Containers (CaaS) and functions (FaaS) are where real decisions get made today. "IaaS versus PaaS" is a useful frame for a whiteboard; "ECS versus Lambda versus a plain instance" is the actual conversation.

---

## 5. Managed services are PaaS wearing different clothes

```text
You need a database.

IaaS approach   → an instance + PostgreSQL + backups + failover + patching
PaaS approach   → RDS: choose a size, get an endpoint     ← see RDS
```

> [!IMPORTANT]
> **A managed database is the highest-value trade in this whole topic.** The work you hand over — point-in-time recovery, automated failover, version upgrades, replica setup — is difficult, unglamorous, and catastrophic when done badly. Paying roughly double the raw instance cost to remove it from your on-call rotation is one of the few decisions almost nobody regrets. See [RDS](RDS.md).

---

## 6. The cost inversion

```text
Per unit of compute:      IaaS < CaaS < PaaS < FaaS
Per unit of engineer time: IaaS > CaaS > PaaS > FaaS
```

```text
At low and spiky volume     → higher-level wins on total cost
At high and steady volume   → lower-level wins on total cost
The crossover is real, and it arrives later than people expect
```

> [!CAUTION]
> **Comparing only the infrastructure invoice is how teams talk themselves into work they cannot afford.** A platform that costs three times the raw instance price but removes half an engineer's week is cheaper by any honest measure. Compare *total* cost, and include the incident you will have at 3 a.m. because nobody set up the standby correctly.

---

## 7. Lock-in, honestly

```text
IaaS   a Linux box is a Linux box                      → low lock-in
CaaS   a container runs anywhere                       → low-ish
PaaS   the platform's build and config are its own     → moderate
FaaS   the event model and glue are provider-specific  → high
SaaS   your data is inside their product               → highest
```

> [!TIP]
> **Lock-in is a price, not a sin — pay it deliberately.** The question is not "could we leave?" but "how many weeks would leaving cost, and is that acceptable for what we gain?" Two weeks of migration in exchange for two years of not operating a database is an easy trade. Where lock-in genuinely bites is data gravity: moving terabytes out is slow and the egress bill is unpleasant.

---

## 8. Real World Example

- **A startup MVP** — PaaS or FaaS, because engineer time is the scarcest resource.
- **A regulated bank** — IaaS or private cloud, because control is a requirement, not a preference.
- **A high-volume video encoder** — IaaS with spot instances, because compute dominates the bill.
- **An internal admin tool** — SaaS, always; nobody should build a helpdesk.
- **A mature product team** — usually CaaS: containers on a managed scheduler, which is the pragmatic middle.

---

## 9. Communication and Dependencies

- **Your team's actual operational capacity** — the real input to this decision
- **[IAM](IAM.md)** — permissions apply at every level
- **[Monitoring and Alerting](../14%20-%20Scalability%20and%20Reliability/Monitoring%20and%20Alerting.md)** — the higher the level, the more you rely on the provider's view
- **A deployment mechanism** — see [CI CD](../13%20-%20DevOps%20and%20Delivery/CI%20CD.md)
- **A billing model** you have read carefully

---

## 10. When To Use / When NOT To Use

> [!TIP]
> **Default to the highest level that does not block you.** Start at SaaS, drop to PaaS when you need custom code, drop to containers when you need control over the runtime, and drop to raw instances only when something specific forces you there.

> [!CAUTION]
> - **Do not choose IaaS for control you will not use** — an unpatched instance is worse than a managed platform
> - **Do not choose FaaS for long, steady, CPU-heavy work** — you will pay a premium for the wrong shape
> - **Do not choose Kubernetes for three services** — the cluster becomes the project
> - **Do not build what SaaS already sells** unless it is your actual product
> - **Do not mix five models** in one small system because each looked best in isolation

---

## 11. Advantages and Disadvantages

**Advantages of moving up the stack**
- Far less operational surface to own
- Security patching becomes someone else's job
- Faster from idea to production
- Scaling behaviour comes built in
- A small team can run a serious system

**Disadvantages of moving up the stack**
- Higher unit cost at scale
- Constraints you cannot work around — timeouts, runtimes, sizes
- Less visibility when something is slow
- Stronger lock-in, especially at FaaS and SaaS
- You inherit the provider's outages and their opinions

---

## 12. Performance Impact

| Aspect | Impact |
|---|---|
| **IaaS** | Full control; you can tune the kernel and the disk if you know how |
| **CaaS** | Near-native, with scheduler overhead and noisier neighbours |
| **PaaS** | Good enough, but the tuning knobs you want may not exist |
| **FaaS** | Cold starts and execution limits become architectural constraints |
| **SaaS** | Entirely their problem, and entirely outside your control |

> [!CAUTION]
> **Higher levels remove your ability to fix a performance problem.** On an instance, a slow query is something you can profile and index. On a platform, you may only see that a request took 900 ms and be left filing a support ticket. That loss of debuggability is the least discussed cost of moving up.

---

## 13. Security Considerations

> [!CAUTION]
> **Moving up the stack shrinks your attack surface but concentrates your risk in identity and configuration.** With IaaS you can be breached through an unpatched OS. With SaaS you cannot — you get breached through a misconfigured sharing setting or a compromised account instead. Neither is safer in the abstract; they fail differently.

- **IaaS** — patching, hardening and network rules are yours; see [Cloud Security](Cloud%20Security.md)
- **PaaS / FaaS** — the runtime is patched for you, but **your dependencies are not**
- **SaaS** — vendor due diligence, SSO enforcement, and reviewing what data leaves your systems
- **Every level** — least privilege, no shared credentials, audit logs on; see [IAM](IAM.md)
- **A vendor's breach is your breach** as far as your customers are concerned

---

## 14. Mental Model

> [!NOTE]
> **Getting dinner.**
>
> **IaaS** is a kitchen — you buy ingredients, cook, and clean up. **PaaS** is a meal kit — the shopping and portioning are done, you still cook. **FaaS** is a food court — you order one dish, it arrives in minutes, and you eat it their way. **SaaS** is a restaurant. Nobody cooks every meal from raw ingredients, and nobody eats out every night. Most engineering organisations should live on meal kits with a functioning kitchen for the few things that matter.

---

## 15. Mini Architecture Diagram

```text
        SaaS   ┌─────────────────────────────┐  provider runs all of it
               │ finished product            │
        FaaS   ├─────────────────────────────┤
               │ your function               │  ← you write only this
        PaaS   ├─────────────────────────────┤
               │ your application            │  ← and this
               │ ─────────────────────────── │
        CaaS   │ your container image        │  ← and this
               │ ─────────────────────────── │
        IaaS   │ your OS, runtime, patching  │  ← and ALL of this
               │ ─────────────────────────── │
               │ virtualisation              │
               │ hardware, power, cooling    │  provider, always
               └─────────────────────────────┘
```

---

## 16. Complete Request Flow

```text
The same feature, at three levels
───────────────────────────────────────────────

IaaS
    provision an instance → harden the OS → install the runtime
    → configure a reverse proxy → write a deploy script
    → set up log shipping → configure backups → THEN write the feature

PaaS
    push the repository → the platform builds and runs it
    → logs and metrics are already there → write the feature

FaaS
    write the handler → deploy → the platform scales it to zero and back
    → accept the timeout and cold-start behaviour as constraints

───────────────────────────────────────────────
Two years later
    IaaS  team owns patching, upgrades, capacity — and can tune anything
    PaaS  team owns the code — and is stuck when the platform lacks a knob
    FaaS  team owns the functions — and is rewriting the ones that grew too big
```

---

## 17. Key Takeaway

> [!IMPORTANT]
> Pick the highest level that does not block your requirements, and judge the choice by total cost — including engineer time and on-call load — not by the infrastructure invoice alone.

---

## 18. Common Mistakes

- **Choosing IaaS for control the team never exercises**, so nothing is patched
- **Comparing only the compute bill** and ignoring the salary cost of running it
- **Adopting Kubernetes far too early**, making the platform the product
- **Assuming PaaS means no operations** — you still own dependencies, migrations and cost
- **Treating lock-in as forbidden** rather than as a price with a number attached
- **Building an internal version of something SaaS sells** for less than one engineer-month
- **Mixing many models in one small system**, so nobody can explain the deployment

---

## 19. Open Source Technologies

- **Kubernetes**, **Nomad** — self-hosted container platforms
- **Dokku**, **CapRover**, **Coolify** — small self-hosted PaaS on your own instance
- **OpenFaaS**, **Knative** — functions without a specific provider
- **Terraform / OpenTofu** — provision at any level; see [Terraform](../13%20-%20DevOps%20and%20Delivery/Terraform.md)
- **Docker** — the artefact that makes moving between levels realistic

---

## 20. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 21. Workbook Exercise

- [ ] List every layer of your current system and write a name beside each one — who is on call for it?
- [ ] Pick one layer you operate yourself and price the managed equivalent.
- [ ] Estimate, in weeks, what leaving your most locked-in service would cost. Decide whether that is acceptable.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
hardware → virtualisation → OS → runtime → application → data
              ↑ provider's side          you draw the line somewhere here
```

## 2. Request Flow

```text
Input       a requirement, plus an honest count of your operational capacity
    ↓
Processing  choose where the provider's responsibility ends and yours begins
    ↓
Output      a system whose unit cost, flexibility and on-call load all follow from that choice
```

## 3. Real-World Usage

The industry's centre of gravity has settled on **containers on a managed scheduler with managed data stores** — high enough to avoid patching servers, low enough to keep control of the runtime. Functions win for glue and spiky work; raw instances survive where compute cost or hardware access dominates.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Three names for how much of the stack the provider runs |
| **Why does it exist?** | Because most layers of a system are nobody's competitive advantage |
| **Where does it belong?** | It is a decision about every layer, not a component in one |
| **When should I use it?** | Always — choose the highest level your requirements allow |
