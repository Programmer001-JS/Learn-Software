# AWS Architecture

> **In one line —** the market's default cloud, understood best not as two hundred services but as five building blocks and one authorisation system that governs all of them.

| | |
|---|---|
| **Full name** | Amazon Web Services |
| **Category** | Overview note *(hub for this sub-section)* |
| **Architectural Layer** | Infrastructure |
| **Sub-topics** | [EC2](EC2.md) · [S3](S3.md) · [RDS](RDS.md) · [Lambda](Lambda.md) · [VPC](VPC.md) · [IAM](IAM.md) |
| **Related notes** | [Cloud Fundamentals](Cloud%20Fundamentals.md) · [AWS SQS](../10%20-%20Distributed%20Systems/AWS%20SQS.md) · [Terraform](../13%20-%20DevOps%20and%20Delivery/Terraform.md) · [Cloud Security](Cloud%20Security.md) |

---

## 1. Short Definition

*What is it?*

AWS is a **collection of independent, API-driven services** sharing one account model, one identity system and one regional structure. It is not a product you learn — it is a catalogue you learn to navigate.

---

## 2. Problem

*What engineering problem does it solve?*

```text
AWS offers 200+ services
    ↓
Six of them appear in nearly every architecture
    ↓
The rest exist for specific problems you may never have
    ↓
The difficulty is not learning services — it is knowing which six
```

> [!IMPORTANT]
> **The service count is a marketing artefact, not a learning requirement.** A production system for most companies uses compute, object storage, a managed database, a queue, a load balancer and identity. If you know those, you can read any AWS architecture diagram. Everything else you can look up on the day you need it.

---

## 3. Architecture Position

```text
ACCOUNT             the billing and isolation boundary
    │
    ├── IAM         who may do what — governs EVERY call below
    │
    ├── REGION      eu-central-1
    │     ├── VPC             your private network
    │     │     ├── AZ-a subnets → EC2, RDS primary
    │     │     └── AZ-b subnets → EC2, RDS standby
    │     ├── S3             regional, outside the VPC
    │     ├── SQS / SNS      regional, outside the VPC
    │     └── Lambda         runs in AWS's network, or inside your VPC
    │
    └── GLOBAL      IAM, Route 53, CloudFront, billing
```

> [!TIP]
> **Notice what is inside the VPC and what is not.** EC2 and RDS live in your network. S3, SQS, Lambda and DynamoDB are regional services reached over the AWS network by API call. This single distinction explains most confusion about AWS networking — including why a private instance can still reach S3 (via a gateway endpoint) and why "is it in my VPC?" is the first question in any connectivity problem.

---

## 4. The services that actually matter

| Service | What it is | Note |
|---|---|---|
| **[IAM](IAM.md)** | Identity and permissions | Learn first. It gates everything |
| **[VPC](VPC.md)** | Your private network | The second thing to learn |
| **[EC2](EC2.md)** | Virtual machines | The oldest primitive; still everywhere |
| **[S3](S3.md)** | Object storage | Effectively unlimited, extremely durable |
| **[RDS](RDS.md)** | Managed relational databases | The highest-value managed service |
| **[Lambda](Lambda.md)** | Functions | Glue, events, spiky work |
| **ALB / NLB** | Load balancers | Application (L7) and network (L4) |
| **[SQS](../10%20-%20Distributed%20Systems/AWS%20SQS.md) / SNS / EventBridge** | Queues, fan-out, events | Decoupling |
| **CloudWatch** | Metrics, logs, alarms | Adequate; see [Cloud Monitoring](Cloud%20Monitoring.md) |
| **Route 53** | DNS | Also health-check-based failover |
| **CloudFront** | CDN | And the cheapest way to cut egress cost |
| **ECS / EKS** | Containers | ECS for simplicity, EKS for Kubernetes |
| **DynamoDB** | Managed key-value store | Superb if the access pattern fits, painful if not |
| **KMS / Secrets Manager** | Keys and secrets | Not optional in production |

---

## 5. The account structure decision

```text
ONE ACCOUNT           simple, and a mistake at any real size
    ↓
MULTI-ACCOUNT (AWS Organizations)
    management account   →  billing and guardrails only, nothing runs here
    prod account         →  production, tightly locked down
    staging account      →  a real copy, safe to break
    dev / sandbox        →  per-team, with a spending cap
    log archive          →  write-only audit trail
```

> [!IMPORTANT]
> **An account is AWS's strongest isolation boundary — stronger than any permission policy.** A mistake in a dev account cannot delete a production database if production lives in a different account, and no IAM policy is needed to guarantee it. Separating production from everything else is the highest-value structural decision you will make on AWS, and it is far easier before you have resources than after.

---

## 6. The paths a request can take

```text
PUBLIC HTTP TRAFFIC
    Route 53 → CloudFront → ALB → ECS/EC2 → RDS
                    │
                    └→ S3 for static assets and media

EVENT-DRIVEN WORK
    S3 upload → EventBridge → Lambda → SQS → worker → RDS

INTERNAL SERVICE CALLS
    service → Cloud Map / ALB → service → DynamoDB
```

> [!TIP]
> **Almost every AWS architecture is one of these three shapes, composed.** When you look at an unfamiliar diagram, find the request path first, then the event path, then the data stores. The remaining boxes are almost always observability, secrets or CI.

---

## 7. Where the money goes

```text
Typical bill, roughly ordered
    EC2 / ECS compute        often the largest single item
    DATA TRANSFER OUT        frequently second, and usually a surprise
    RDS                      including the standby you are paying for
    S3                       storage is cheap; request volume is not always
    NAT Gateway              charged per hour AND per GB processed
    CloudWatch Logs          ingestion cost grows silently with debug logging
```

> [!CAUTION]
> **Two line items catch out nearly every team: the NAT Gateway and CloudWatch Logs.** A NAT Gateway costs a fixed hourly rate plus a per-gigabyte processing charge, so a private subnet pulling container images through it can quietly cost more than the instances themselves — use VPC endpoints for S3 and ECR. CloudWatch Logs bills on ingestion, so one service left at debug level can outrun your compute bill. Both are invisible until you read the bill by service.

---

## 8. Real World Example

- **Netflix** — the largest and most documented AWS architecture; see [Netflix Architecture](../15%20-%20Real%20World%20System%20Design/Netflix%20Architecture.md).
- **Amazon's own retail platform** — see [Amazon Architecture](../15%20-%20Real%20World%20System%20Design/Amazon%20Architecture.md).
- **A typical SaaS** — ALB, ECS Fargate, RDS Multi-AZ, S3, SQS, CloudFront. Six services, and it scales a long way.
- **Data pipelines** — S3 as the lake, Glue or EMR for processing, Athena for queries.
- **Media processing** — S3 events triggering Lambda, then containers for the heavy encoding; see [Video Processing Platform](../15%20-%20Real%20World%20System%20Design/Video%20Processing%20Platform.md).

---

## 9. Communication and Dependencies

- **Everything depends on [IAM](IAM.md)** — a call without permission never reaches the service
- **Regional coupling** — most services are regional; design for one region first
- **[VPC](VPC.md)** for anything with a network interface
- **[Terraform](../13%20-%20DevOps%20and%20Delivery/Terraform.md)** or CloudFormation — the console is for reading, not creating
- **us-east-1** — some global services depend on it, which is why its outages feel wider than they are

---

## 10. When To Use / When NOT To Use

> [!TIP]
> AWS is the safe default: the widest service catalogue, the deepest documentation, and the largest pool of engineers who already know it. Hiring and troubleshooting are both easier here than anywhere else.

> [!CAUTION]
> - **Not if a simpler host is enough** — a small application on a $20 VPS or a PaaS does not need an AWS account, a VPC and six IAM roles
> - **Not if nobody on the team will own cost** — AWS will happily bill you for forgotten resources indefinitely
> - **Not for the newest managed AI or data tooling by default** — competitors are sometimes genuinely ahead
> - **Not by clicking in the console** — untracked resources are the beginning of an unmaintainable account

---

## 11. Advantages and Disadvantages

**Advantages**
- The most complete service catalogue available
- Mature, stable APIs with strong backwards compatibility
- Excellent regional and AZ coverage
- The largest talent pool and community
- Fine-grained permissions, if you invest in them
- Almost any architecture can be built without leaving the platform

**Disadvantages**
- **Overwhelming surface area**, with several overlapping ways to do everything
- Pricing is genuinely difficult to predict
- Poor defaults in places — public access, wide permissions, no log retention limit
- IAM is powerful and unpleasant to learn
- Support costs extra to be useful
- Console UX varies wildly between services
- Some services are effectively abandoned but never removed

---

## 12. Performance Impact

| Aspect | Impact |
|---|---|
| **Within an AZ** | Sub-millisecond; treat as local |
| **Across AZs** | 1–2 ms, and charged per GB in both directions |
| **Across regions** | Tens to hundreds of milliseconds |
| **S3 first byte** | Tens of milliseconds; parallelise for throughput |
| **Lambda cold start** | Milliseconds to seconds depending on runtime and VPC config |
| **API throttling** | Hard limits per service; retry with backoff or fail |

> [!TIP]
> **Cross-AZ data transfer is charged in both directions and it adds up in chatty microservice architectures.** It is not a reason to run in a single AZ — availability is worth more — but it is a reason to keep high-volume conversations, such as an application and its cache, zone-aware.

---

## 13. Security Considerations

> [!CAUTION]
> **The three failures that account for most AWS incidents, in order:** a public S3 bucket, an over-permissive IAM role or leaked long-lived access key, and a security group open to `0.0.0.0/0`. None of them are subtle, all of them are entirely preventable, and all of them are still happening this year.

- **Root account** — MFA on, access keys deleted, used almost never
- **No long-lived access keys** — use roles for services and SSO for humans; see [IAM](IAM.md)
- **CloudTrail on in every region**, delivered to a separate log account
- **S3 Block Public Access at the account level**, then justify every exception; see [S3](S3.md)
- **Private subnets by default**, security groups referencing each other rather than CIDRs; see [VPC](VPC.md)
- **Secrets in Secrets Manager or Parameter Store**, never in environment variables committed to a repository
- **GuardDuty and Config** — cheap detection that catches the obvious mistakes
- See [Cloud Security](Cloud%20Security.md) for the full treatment

---

## 14. Mental Model

> [!NOTE]
> **AWS is a hardware catalogue with an API and a very strict receptionist.**
>
> The catalogue is enormous, and most of it is not for you. Every request — from launching a thousand servers to reading one file — goes through the same receptionist, IAM, who checks whether you are allowed. Learn the receptionist's rules and the catalogue becomes browsable. Skip them and nothing works, for reasons the error message will not explain.

---

## 15. Mini Architecture Diagram

```text
                Route 53 (DNS, global)
                        │
                CloudFront (CDN, edge)
                        │
        ┌───────── REGION: eu-central-1 ──────────┐
        │                                          │
        │   ┌──── VPC 10.0.0.0/16 ─────────────┐  │
        │   │  PUBLIC SUBNETS                   │  │
        │   │    ALB (AZ-a, AZ-b)               │  │
        │   │    NAT Gateway                    │  │
        │   │  ─────────────────────────────    │  │
        │   │  PRIVATE SUBNETS                  │  │
        │   │    ECS tasks / EC2 (AZ-a, AZ-b)   │  │
        │   │    RDS primary ──sync──> standby  │  │
        │   │    ElastiCache                    │  │
        │   └───────────┬───────────────────────┘  │
        │               │ VPC endpoints             │
        │   S3 · SQS · Secrets Manager · ECR        │
        └───────────────────────────────────────────┘
                        │
            IAM + CloudTrail wrap all of it
```

---

## 16. Complete Request Flow

```text
DNS lookup at Route 53 → nearest CloudFront edge
    ↓
Cache hit → returned from the edge, no origin traffic, no regional egress
    ↓
Cache miss → forwarded to the ALB in eu-central-1
    ↓
ALB terminates TLS, picks a healthy ECS task (AZ-a or AZ-b)
    ↓
The task assumes an IAM role — no keys anywhere in the image
    ↓
Reads from RDS primary in a private subnet (no route to the internet)
    ↓
Writes a media file to S3 via a gateway endpoint (no NAT charge)
    ↓
Publishes a message to SQS; a worker picks it up asynchronously
    ↓
Response returns through the ALB and CloudFront to the user
    ↓
─────────── in the background ───────────
CloudTrail records every API call to the log archive account
CloudWatch alarms watch error rate and queue depth
RDS takes automated backups; S3 versions the object
```

---

## 17. Key Takeaway

> [!IMPORTANT]
> Learn IAM and VPC first, then six services, then separate production into its own account — that is 90% of AWS competence, and the remaining 194 services can wait until a problem asks for them.

---

## 18. Common Mistakes

- **Trying to learn AWS broadly** instead of learning six services deeply
- **Everything in one account**, with development able to reach production
- **Clicking resources into existence** and never being able to reproduce the environment
- **Ignoring the NAT Gateway and CloudWatch Logs bills** until they are the largest items
- **Running in a single AZ** because the second one looked like unnecessary cost
- **Long-lived IAM access keys** instead of roles
- **Trusting default settings** — several are convenient rather than safe
- **Choosing DynamoDB before knowing the access patterns**, then needing a join
- **Assuming us-east-1 is just another region** when several global services depend on it

---

## 19. Open Source Technologies

- **Terraform / OpenTofu**, **Pulumi**, **AWS CDK** — infrastructure as code
- **LocalStack** — run AWS APIs locally for testing
- **aws-vault**, **Leapp** — handle credentials without storing keys on disk
- **Prowler**, **ScoutSuite**, **cloudsplaining** — audit an account for misconfiguration
- **Steampipe** — query your AWS account with SQL
- **Infracost** — estimate the cost of a change before merging it

---

## 20. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 21. Workbook Exercise

- [ ] Draw your current architecture using only the boxes in section 15. Note what is missing.
- [ ] Open Cost Explorer, group by service, and identify your top three line items.
- [ ] Check whether any S3 bucket in your account allows public access.
- [ ] Confirm CloudTrail is enabled and that its logs live somewhere you cannot delete casually.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Account → IAM → Region → VPC (EC2, RDS) + regional services (S3, SQS, Lambda)
```

## 2. Request Flow

```text
Input       an authenticated API call or an HTTP request at the edge
    ↓
Processing  IAM authorises, the VPC routes, the service does the work
    ↓
Output      a result, a CloudTrail entry, and a line on next month's bill
```

## 3. Real-World Usage

The architecture that keeps appearing in production — load balancer, containers across two AZs, a Multi-AZ managed database, object storage, a queue, and a CDN in front — is boring on purpose. It is six services, it survives a data centre failure, and it carries most companies further than they expect before anything more exotic is needed.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A catalogue of independent API-driven services under one account and identity model |
| **Why does it exist?** | Because renting composable infrastructure beat building it |
| **Where does it belong?** | Underneath the whole system |
| **When should I use it?** | As the default cloud — unless a simpler host genuinely suffices |
