# EC2

> **In one line —** a virtual machine you rent by the second; the oldest and least magical AWS service, and still the right answer more often than fashion suggests.

| | |
|---|---|
| **Full name** | Amazon Elastic Compute Cloud |
| **Category** | IaaS Compute |
| **Architectural Layer** | Infrastructure |
| **Related notes** | [AWS Architecture](AWS%20Architecture.md) · [VPC](VPC.md) · [IaaS PaaS SaaS](IaaS%20PaaS%20SaaS.md) · [Lambda](Lambda.md) · [Horizontal Scaling](../14%20-%20Scalability%20and%20Reliability/Horizontal%20Scaling.md) · [Docker](../13%20-%20DevOps%20and%20Delivery/Docker.md) |

---

## 1. Short Definition

*What is it?*

EC2 gives you a **virtual server** — CPU, memory, network interface and disk — running an operating system you choose, inside a network you control. You get root, and with it every responsibility that root implies.

---

## 2. Problem

*What engineering problem does it solve?*

```text
You need a server
    ↓
Buying one:      weeks of lead time, thousands up front, yours for five years
Renting one:     an API call, running in 40 seconds, billed per second
    ↓
Wrong size?  →  stop it, change the type, start it
Not needed?  →  terminate it, billing stops
```

> [!IMPORTANT]
> **EC2's contribution was not virtualisation — VMware had that. It was making a server disposable.** Once a machine can be destroyed and recreated in a minute, you stop repairing servers and start replacing them. Every modern practice — immutable infrastructure, auto scaling, blue/green deployment — depends on that shift.

---

## 3. Architecture Position

```text
        VPC (your network)
            │
    ┌───────┴────────┐
    │  SUBNET (AZ-a) │
    │   ┌──────────┐ │
    │   │   EC2    │ │  ← an OS you patch, a runtime you install
    │   │ instance │ │
    │   └────┬─────┘ │
    │        │ ENI    │  network interface, security group attached
    │        │ EBS    │  network-attached disk, survives a stop
    └────────┼────────┘
             │
    Load balancer in front · Auto Scaling Group managing the count
```

---

## 4. Instance families

```text
t   BURSTABLE     cheap, accrues CPU credits    → dev, low traffic, small services
m   GENERAL       balanced CPU:memory (1:4)     → the honest default
c   COMPUTE       CPU-heavy                     → encoding, simulation, build farms
r   MEMORY        lots of RAM (1:8)             → caches, in-memory analytics, big JVMs
i   STORAGE       fast local NVMe               → databases you run yourself
g/p GPU           accelerators                  → training and inference
```

```text
Naming:  m7g.xlarge
         │││ └── size — each step up doubles CPU and memory, and the price
         ││└──── g = Graviton (ARM); nothing = Intel; a = AMD
         │└───── generation — newer is usually cheaper per unit of work
         └────── family
```

> [!TIP]
> **Try Graviton before you try optimising code.** ARM instances are typically 20–40% cheaper for the same throughput, and for interpreted or JVM workloads the migration is often just rebuilding the container for `arm64`. It is one of the rare changes with a large cost win and near-zero architectural risk.

> [!CAUTION]
> **`t` instances throttle, and it does not look like throttling.** Burstable types earn CPU credits while idle and spend them under load. When credits run out, the instance is capped at its baseline — sometimes 10–20% of a core — and your application simply becomes inexplicably slow while CPU metrics look calm. Fine for development, dangerous for anything with sustained traffic.

---

## 5. The four purchasing models

| Model | Discount | The catch |
|---|---|---|
| **On-demand** | none | Pay full price for total flexibility |
| **Savings Plans / Reserved** | up to ~70% | A 1–3 year spend commitment |
| **Spot** | up to ~90% | **Can be reclaimed with two minutes' warning** |
| **Dedicated hosts** | negative | You pay more, for licensing or compliance reasons |

> [!IMPORTANT]
> **Spot instances are the single largest cost lever in AWS, and most teams never touch them.** Any workload that is interruptible and retryable — CI runners, batch jobs, video encoding, stateless web tiers behind a load balancer — can run at a fraction of the price. The engineering requirement is genuinely small: handle the two-minute termination notice and make the work resumable. Do that once and the saving is permanent.

---

## 6. Storage: EBS versus instance store

```text
EBS (network-attached)
    survives stop/start and instance replacement
    snapshot-able, resizable, encrypted with one flag
    gp3 → the default; io2 → when you genuinely need guaranteed IOPS
    ↓
    It is network storage. Latency and throughput have limits.

INSTANCE STORE (physical NVMe on the host)
    very fast
    DATA IS GONE when the instance stops — not just when it terminates
    → caches, scratch space, replicated databases only
```

> [!CAUTION]
> **On gp3 you provision IOPS and throughput independently of size, and the defaults are modest.** A database on a default gp3 volume can be limited by disk long before CPU or memory look busy. Check the EBS burst-balance and throughput metrics before concluding your application is slow — it is a routine misdiagnosis.

---

## 7. Auto Scaling Groups: the part that matters

```text
LAUNCH TEMPLATE       what an instance should be — AMI, type, user data, role
    +
AUTO SCALING GROUP    how many, in which AZs, and what "healthy" means
    +
TARGET TRACKING       "keep average CPU near 50%"
    ↓
Unhealthy instance → terminated and replaced, no human involved
Traffic rises       → more instances, spread across AZs
Traffic falls       → instances removed
```

> [!TIP]
> **The ASG's real value is self-healing, not scaling.** Most systems have predictable load and rarely scale in a day — but every system eventually has an instance that hangs, fills its disk or fails a health check. An ASG replaces it automatically at 4 a.m. Run even a fixed-size fleet inside one, with a minimum of two across two AZs.

> [!CAUTION]
> **Scaling only helps if new instances are ready quickly.** If your AMI needs six minutes of package installation and application setup on boot, your response to a traffic spike arrives after the spike. Bake dependencies into the image, or run containers — see [Docker](../13%20-%20DevOps%20and%20Delivery/Docker.md).

---

## 8. Cattle, not pets

```text
PET                             CATTLE
named server                    numbered, interchangeable instance
patched in place                replaced with a new image
SSH in to fix things            terminate it; the ASG makes another
state lives on the disk         state lives in RDS, S3 or a cache
6 months of undocumented drift  identical to every sibling
```

> [!IMPORTANT]
> **If you cannot terminate any single instance right now without consequences, you do not have cloud infrastructure — you have a data centre with an API.** This is the test. Everything else about EC2 is a detail; this is the discipline that makes the rest work.

---

## 9. Real World Example

- **Web and application tiers** behind an ALB, in an ASG across two AZs — the standard shape.
- **CI build farms** on Spot, where an interruption costs one retried build.
- **Self-managed databases** on `i` or `r` instances where RDS lacks a needed extension or version.
- **GPU training and inference** on `p` and `g` families; see [GPU Computing](../11%20-%20AI%20Engineering/GPU%20Computing.md).
- **Lift-and-shift migrations**, because an old application usually just needs a Linux box.
- **Kubernetes nodes** — an EKS cluster's workers are EC2 instances; see [Kubernetes](../13%20-%20DevOps%20and%20Delivery/Kubernetes.md).

---

## 10. Communication and Dependencies

- **[VPC](VPC.md), subnet and security group** — an instance cannot exist outside a network
- **An IAM instance role** — how the instance gets credentials without storing keys; see [IAM](IAM.md)
- **An AMI** — the image, ideally built by a pipeline rather than by hand
- **EBS volumes**, and snapshots as your backup path
- **A load balancer** for anything serving traffic
- **A patching mechanism** — the OS is yours now

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Use EC2 when you need control of the operating system, a specific runtime or kernel setting, GPU or local NVMe hardware, long-running processes, or predictable compute cost at steady load. It is also the correct pragmatic answer for migrating something old.

> [!CAUTION]
> - **Not for short, spiky, event-driven work** — [Lambda](Lambda.md) is a better fit and cheaper
> - **Not for a managed database's job** — [RDS](RDS.md) does failover and backups properly
> - **Not if nobody will patch it** — an unmaintained instance becomes a liability within months
> - **Not as a pet** with state on the local disk and configuration applied by hand
> - **Not without an ASG**, even at a fixed size

---

## 12. Advantages and Disadvantages

**Advantages**
- Complete control of the OS, runtime and kernel
- Every instance shape imaginable — CPU, memory, GPU, local NVMe
- Cheapest per unit of compute at steady load, especially with Spot and Graviton
- No cold starts and no execution time limits
- Anything that runs on Linux or Windows runs here
- Mature, predictable, and unlikely to change under you

**Disadvantages**
- **You own patching, hardening and configuration drift**
- Minutes to provision, not milliseconds
- You pay for idle capacity
- Scaling must be designed, not assumed
- Every instance is an attack surface you maintain
- Easy to accumulate forgotten instances that bill forever

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Boot time** | 30–90 seconds to reachable, plus whatever your bootstrap does |
| **Network** | Up to 100 Gbit/s on large types; small types are capped and throttle |
| **EBS** | Network storage — gp3 defaults to modest IOPS and throughput |
| **Instance store** | Local NVMe, dramatically faster, ephemeral |
| **`t` family** | Throttled to baseline once CPU credits are exhausted |
| **Placement groups** | Cluster placement lowers inter-node latency for tight workloads |

> [!TIP]
> **When an EC2 instance is mysteriously slow, check the credit and burst-balance metrics before the application.** CPU credits on `t` types and EBS burst balance on small volumes cause exactly the symptom people attribute to code: normal-looking utilisation, terrible latency.

---

## 14. Security Considerations

> [!CAUTION]
> **Enforce IMDSv2 and never put credentials on an instance.** The instance metadata service hands out the role's temporary credentials to anything that can make a local HTTP request — which historically included a server-side request forgery bug in your own application. IMDSv2 requires a session token and blocks that class of attack. The 2019 Capital One breach was, in essence, this path.

- **No SSH keys and no open port 22** — use SSM Session Manager, which is logged and needs no inbound rule
- **Security groups reference other security groups**, not `0.0.0.0/0`; see [VPC](VPC.md)
- **Instance roles, never access keys** baked into an AMI or user data
- **Encrypt EBS by default** at the account level — it costs nothing
- **Patch on a schedule** you can prove, or replace instances from fresh images instead
- **Private subnets** for anything not required to be public
- **User data is not secret** — it is readable from inside the instance

---

## 15. Mental Model

> [!NOTE]
> **EC2 is a rental car, and the Auto Scaling Group is the rental company.**
>
> You get the keys and full control — you can load it however you like. You are also responsible for the oil. The important part is that when one breaks down you do not repair it at the roadside; you return it and take another, because the company has a lot of identical cars. Treating a rental car like your own beloved vehicle is exactly the mistake people make with instances.

---

## 16. Mini Architecture Diagram

```text
                   Internet
                       │
            ┌──────────▼───────────┐
            │  ALB (public subnets)│  AZ-a + AZ-b
            └──────────┬───────────┘
                       │
    ┌──────── AUTO SCALING GROUP (min 2) ─────────┐
    │                                              │
    │  PRIVATE SUBNET AZ-a      PRIVATE SUBNET AZ-b│
    │   ┌──────────────┐         ┌──────────────┐  │
    │   │ EC2 instance │         │ EC2 instance │  │
    │   │  IAM role    │         │  IAM role    │  │
    │   │  gp3 EBS     │         │  gp3 EBS     │  │
    │   └──────┬───────┘         └───────┬──────┘  │
    └──────────┼─────────────────────────┼─────────┘
               │                         │
               ▼                         ▼
        RDS (Multi-AZ)            S3 via VPC endpoint
               │
        NAT Gateway → outbound internet only
```

---

## 17. Complete Request Flow

```text
Launch template references AMI + instance type + IAM role + user data
    ↓
ASG launches an instance into a private subnet in AZ-a
    ↓
Instance boots (~45 s), user data starts the container runtime
    ↓
Instance requests credentials from IMDSv2 — temporary, auto-rotating
    ↓
ALB health check passes → the instance starts receiving traffic
    ↓
─────────────── steady state ───────────────
Requests arrive from the ALB, reads go to RDS, media to S3 via endpoint
    ↓
─────────────── the instance degrades ───────────────
Health check fails three times
    ↓
ALB stops routing to it; ASG marks it unhealthy
    ↓
Instance terminated, a replacement launched from the same template
    ↓
No human involved, no state lost — because none was stored locally
    ↓
─────────────── traffic spike ───────────────
Target tracking sees average CPU at 80%
    ↓
Two more instances launched, spread across both AZs
    ↓
Spike passes → scaled back in after the cooldown
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> EC2 gives you full control and the lowest unit cost, in exchange for owning the operating system — treat instances as disposable cattle inside an Auto Scaling Group, and take the free money in Graviton and Spot.

---

## 19. Common Mistakes

- **Treating instances as pets** — SSH in, fix by hand, never reproducible
- **State on the local disk**, so no instance can ever be replaced
- **`t` instances for production traffic**, then debugging a throttle as an application problem
- **No Auto Scaling Group**, so a failed instance stays failed until someone notices
- **Default gp3 settings under a database**, then blaming the query planner
- **Access keys in user data or the AMI** instead of an instance role
- **Port 22 open to the world** rather than using SSM Session Manager
- **IMDSv1 left enabled**, keeping an SSRF-to-credentials path open
- **Never trying Spot** for interruptible work, and paying five times more than necessary
- **Forgotten instances and unattached EBS volumes** billing quietly for years

---

## 20. Open Source Technologies

- **Packer** — build AMIs reproducibly from a definition file
- **Terraform / OpenTofu** — launch templates and ASGs as code
- **Ansible** — configuration management, when you must configure rather than replace
- **cloud-init** — the standard way user data is executed at boot
- **Docker** — makes instances interchangeable and boot-fast
- **node_exporter**, **Prometheus** — real instance metrics beyond CloudWatch's basics
- **Karpenter** — provisions EC2 nodes intelligently for Kubernetes, Spot included

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Pick any running instance and ask honestly: can I terminate this right now? If not, why not?
- [ ] Check whether any production instance is a `t` type, and look at its CPU credit balance.
- [ ] Identify one workload that could run on Spot and estimate the saving.
- [ ] Verify IMDSv2 is required on your instances, and that no security group allows port 22 from `0.0.0.0/0`.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Launch template → ASG → instances in private subnets across AZs → behind an ALB
```

## 2. Request Flow

```text
Input       a launch template describing what an instance should be
    ↓
Processing  the ASG maintains a healthy count across zones, replacing failures
    ↓
Output      interchangeable compute capacity, billed per second
```

## 3. Real-World Usage

Despite fifteen years of higher-level alternatives, EC2 remains the substrate: EKS nodes, ECS EC2 capacity, self-managed databases and GPU fleets are all instances. The pattern that separates teams that operate it well from those that suffer is single and simple — **can any instance be destroyed without a second thought?**

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A virtual machine rented by the second, inside your own network |
| **Why does it exist?** | To make servers disposable rather than purchased |
| **Where does it belong?** | In private subnets, inside an Auto Scaling Group, behind a load balancer |
| **When should I use it?** | When you need OS control, specific hardware, long-running processes, or the lowest unit cost |
