# VPC

> **In one line —** your own private network inside AWS, where the single most important decision is which subnets can reach the internet and which cannot.

| | |
|---|---|
| **Full name** | Amazon Virtual Private Cloud |
| **Category** | Cloud Networking |
| **Architectural Layer** | Infrastructure |
| **Related notes** | [AWS Architecture](AWS%20Architecture.md) · [EC2](EC2.md) · [RDS](RDS.md) · [Cloud Security](Cloud%20Security.md) · [IP Addresses](../04%20-%20Networking%20and%20Internet/IP%20Addresses.md) · [TCP IP](../04%20-%20Networking%20and%20Internet/TCP%20IP.md) |

---

## 1. Short Definition

*What is it?*

A VPC is a **logically isolated network** you define inside an AWS region: your own IP range, your own subnets, your own routing tables, and your own rules about what may talk to what. Nothing enters or leaves except through a path you created.

---

## 2. Problem

*What engineering problem does it solve?*

```text
Cloud instances on a shared, flat network
    ↓
Every machine reachable from every other machine
Every machine reachable from the internet
No segmentation, no boundary, no way to say "database: internal only"
    ↓
One compromised web server = access to everything
```

The VPC restores the boundary a data centre gave you for free — **but as software, so you can create it correctly instead of inheriting whatever the network team built in 2011.**

> [!IMPORTANT]
> **The important consequence is that network topology became reviewable code.** A firewall rule in a data centre lives in a device's configuration and drifts silently for years. A security group in Terraform appears in a pull request. That is a genuine security improvement, and it only pays off if you actually define the network as code rather than clicking through the wizard.

---

## 3. Architecture Position

```text
REGION
  └── VPC  10.0.0.0/16          your address space (65,536 addresses)
        │
        ├── PUBLIC SUBNET  AZ-a  10.0.0.0/24   → route to Internet Gateway
        │     ALB node, NAT Gateway
        ├── PUBLIC SUBNET  AZ-b  10.0.1.0/24
        │
        ├── PRIVATE SUBNET AZ-a  10.0.10.0/24  → route to NAT only
        │     application instances / ECS tasks
        ├── PRIVATE SUBNET AZ-b  10.0.11.0/24
        │
        ├── DATA SUBNET    AZ-a  10.0.20.0/24  → NO internet route at all
        │     RDS primary
        └── DATA SUBNET    AZ-b  10.0.21.0/24
              RDS standby
```

---

## 4. Public versus private is only about the route table

```text
PUBLIC SUBNET   route table contains  0.0.0.0/0 → Internet Gateway
PRIVATE SUBNET  route table contains  0.0.0.0/0 → NAT Gateway
DATA SUBNET     route table contains  no default route at all
```

> [!IMPORTANT]
> **There is no "public" flag on a subnet — a subnet is public if and only if its route table points at an Internet Gateway.** That is the entire mechanism, and understanding it removes most of the mystery from AWS networking. An instance in a "public" subnet with no public IP address still cannot be reached; an instance in a private subnet with a route to an Internet Gateway is not private at all.

---

## 5. The default: almost everything belongs in a private subnet

```text
PUBLIC SUBNETS should contain only:
    load balancers
    NAT gateways
    bastion hosts — and you probably do not need one (use SSM)

EVERYTHING ELSE goes private:
    application servers    ← reached through the load balancer
    databases              ← reached only from the application
    caches, queues, workers
```

> [!CAUTION]
> **The default VPC AWS creates for you has only public subnets, and this is why so many accounts end up with databases exposed to the internet.** Every instance launched into it gets a public address automatically. It exists to make the first tutorial work. Do not build production in it — define your own VPC, and if you launch anything into a public subnet, be able to say why.

---

## 6. Security groups versus NACLs

```text
SECURITY GROUP                       NETWORK ACL
attached to an instance/ENI          attached to a subnet
STATEFUL — a reply is allowed        STATELESS — you must allow both directions
allow rules only                     allow AND deny rules
can reference another SG             CIDR ranges only
evaluated as a whole                 evaluated in numbered order
→ this is your real firewall         → a blunt secondary control
```

> [!TIP]
> **Do your work in security groups and leave NACLs alone.** Security groups are stateful, composable and readable. Their best feature is referencing each other: the database group allows port 5432 *from the application's security group* rather than from an IP range — so the rule stays correct as instances come and go, and it expresses intent rather than addresses. NACLs earn their place only for coarse subnet-wide denials, and misconfigured ones cause outages that are miserable to diagnose because the stateless return traffic fails silently.

---

## 7. The NAT Gateway, and why it appears on your bill

```text
Private instance needs to reach the internet (an API, a package repository)
    ↓
NAT GATEWAY in a public subnet
    ↓
Outbound allowed, inbound impossible
    ↓
COST: an hourly charge PER GATEWAY  +  a charge PER GIGABYTE processed
    ↓
One per AZ for high availability → the charge multiplies
```

> [!CAUTION]
> **The NAT Gateway is regularly a top-three line item, and most of its traffic is avoidable.** Pulling container images, reading from S3 and calling AWS APIs from private subnets all route through it by default and all are billed per gigabyte. **VPC endpoints fix this**: a gateway endpoint for S3 and DynamoDB is free, and interface endpoints for ECR, Secrets Manager and CloudWatch cost less than the NAT traffic they replace. This is one of the highest-return changes available in an AWS account.

---

## 8. VPC endpoints

```text
WITHOUT ENDPOINTS
    private instance → NAT Gateway → internet → S3 API
    (charged per GB, and the traffic leaves your VPC)

WITH A GATEWAY ENDPOINT  (S3, DynamoDB — free)
    private instance → route table entry → S3
    (never leaves the AWS network, no NAT charge)

WITH AN INTERFACE ENDPOINT  (most other services — hourly + per GB, but cheap)
    a private ENI inside your subnet with a DNS name
```

> [!TIP]
> **Endpoints are a security control as much as a cost control.** With a gateway endpoint you can remove the internet route entirely from a subnet that only needs S3 — turning "outbound is restricted" into "outbound is impossible", which is a much stronger statement during an incident.

---

## 9. CIDR planning, done once

```text
CHOOSE A /16 THAT DOES NOT COLLIDE
    10.0.0.0/16   ← everyone's first choice, and every merger's first conflict
    prefer something deliberate:  10.42.0.0/16

RESERVE ROOM
    /24 per subnet = 251 usable addresses (AWS reserves 5)
    3 tiers × 2-3 AZs = 6-9 subnets, and leave gaps for growth

YOU CANNOT SHRINK A VPC LATER
    you can add secondary CIDR blocks, but the original range is permanent
```

> [!CAUTION]
> **Overlapping CIDR ranges are the one VPC mistake you cannot fix with a change request.** Two networks using `10.0.0.0/16` can never be peered, and this surfaces years later during an acquisition, a partner integration or a VPN to an office. Spending ten minutes choosing an unusual range is the cheapest insurance in this note. Also beware small subnets with EKS, where every pod consumes an IP address and a `/24` disappears quickly.

---

## 10. Real World Example

- **The standard three-tier web application** — public ALB, private application tier, isolated data tier, across two AZs.
- **EKS clusters**, where pod networking consumes VPC addresses directly.
- **Hybrid connectivity** — Site-to-Site VPN or Direct Connect back to an office or data centre.
- **Multi-account architectures** — a Transit Gateway connecting shared services to each account's VPC.
- **PCI or HIPAA environments**, where network isolation is an audit requirement rather than a preference.

---

## 11. Communication and Dependencies

- **Subnets in at least two AZs** — required by ALBs and RDS, and required for availability
- **Route tables**, which are what actually define public and private
- **An Internet Gateway** for inbound, a **NAT Gateway** for outbound-only
- **Security groups** — the real access control layer
- **VPC endpoints** for AWS services, to cut cost and remove internet exposure
- **VPC Flow Logs**, which are the only record of what traffic actually occurred

---

## 12. When To Use / When NOT To Use

> [!TIP]
> You always have a VPC on AWS — the choice is whether you designed it. Define your own, with public, private and data tiers across two or three AZs, before you launch anything you intend to keep.

> [!CAUTION]
> - **Do not build in the default VPC** — everything in it is public by default
> - **Do not create a VPC per application** if they need to talk; you will end up with a peering mesh
> - **Do not use a single AZ**, even though it is cheaper on cross-zone transfer
> - **Do not choose 10.0.0.0/16 out of habit** if there is any chance of a future merger or VPN
> - **Do not use tiny subnets** with EKS, where addresses are consumed per pod
> - **Do not build a bastion host** before checking whether SSM Session Manager solves it

---

## 13. Advantages and Disadvantages

**Advantages**
- Real network isolation, defined as reviewable code
- Security groups that reference each other rather than fragile IP lists
- Fine-grained control over every route
- Endpoints keep AWS traffic off the public internet
- Flow logs give a complete record of connections
- Hybrid connectivity to existing networks

**Disadvantages**
- **The CIDR range is permanent** — a planning mistake lives forever
- NAT Gateways are expensive and easy to overlook
- The concept count is high — subnets, route tables, IGW, NAT, endpoints, NACLs, peering, Transit Gateway
- Cross-AZ traffic is charged in both directions
- Poor defaults in the default VPC
- Peering does not transit, so meshes grow quadratically
- Debugging connectivity means checking five separate layers

---

## 14. Performance Impact

| Aspect | Impact |
|---|---|
| **Within an AZ** | Sub-millisecond; treat as local |
| **Across AZs** | 1–2 ms, plus per-GB charges in both directions |
| **NAT Gateway** | Scales well, but every gigabyte is billed |
| **Interface endpoints** | Add a hop; still faster and cheaper than going via NAT |
| **Security group rules** | No measurable throughput cost |
| **Instance network limits** | Bandwidth and packets-per-second scale with instance size |

> [!TIP]
> **When a connection fails inside a VPC, check in this order: security group on the target, route table on the source subnet, NACL, then DNS.** Nearly every failure is one of the first two. A dropped packet with no reply is almost always a security group or a stateless NACL rule; a connection that resolves to a public address unexpectedly is usually a missing private DNS setting on an endpoint.

---

## 15. Security Considerations

> [!CAUTION]
> **`0.0.0.0/0` on any port other than 80 and 443 on a load balancer is a finding, not a configuration.** Port 22 open to the world, a database security group allowing all sources, an internal admin interface reachable publicly — these are the routine causes of cloud compromise. Security groups should reference other security groups, and the exceptions should be short, deliberate and reviewed.

- **Private by default** — a public IP address requires a justification
- **Security group to security group**, not CIDR ranges, for internal traffic
- **No inbound SSH** — use SSM Session Manager, which is logged and needs no open port
- **Flow logs enabled**, at least on rejected traffic, and shipped somewhere durable
- **Endpoints instead of NAT** where possible, so sensitive subnets have no internet path at all
- **Separate accounts, not just separate subnets**, for the strongest isolation; see [Cloud Security](Cloud%20Security.md)
- **Egress filtering matters too** — a compromised instance exfiltrates outbound, and an unrestricted NAT route is how

---

## 16. Mental Model

> [!NOTE]
> **A VPC is an office building.**
>
> The Internet Gateway is the front door. Public subnets are the lobby — the reception desk and the post room belong there, and nothing else. Private subnets are the offices upstairs: staff can go out through the lobby, but visitors cannot walk in. The data subnets are the archive in the basement, with no door to the outside at all. Security groups are the badge readers on each room; NACLs are the barrier at the corridor entrance. Most cloud breaches are the equivalent of leaving the basement archive accessible from the street.

---

## 17. Mini Architecture Diagram

```text
                        Internet
                            │
                    Internet Gateway
                            │
  ┌───────────── VPC 10.42.0.0/16 ───────────────────────────┐
  │                                                           │
  │  PUBLIC 10.42.0.0/24 (AZ-a)   PUBLIC 10.42.1.0/24 (AZ-b) │
  │    ALB node                     ALB node                  │
  │    NAT Gateway ◄──────┐         NAT Gateway               │
  │                       │                                   │
  │  ─────────────────────┼─────────────────────────────────  │
  │  PRIVATE 10.42.10.0/24│        PRIVATE 10.42.11.0/24      │
  │    ECS tasks ─────────┘ outbound only                     │
  │       │  sg-app                    ECS tasks              │
  │  ─────┼───────────────────────────────────────────────    │
  │  DATA 10.42.20.0/24            DATA 10.42.21.0/24         │
  │    RDS primary  ◄── sg-db allows 5432 FROM sg-app ONLY    │
  │                 ──sync──►  RDS standby                    │
  │    (no default route — cannot reach the internet at all)   │
  │                                                           │
  │  VPC endpoints: S3, DynamoDB (gateway, free)               │
  │                 ECR, Secrets Manager, Logs (interface)     │
  └───────────────────────────────────────────────────────────┘
```

---

## 18. Complete Request Flow

```text
User request → DNS → ALB's public address in a public subnet
    ↓
ALB security group allows 443 from 0.0.0.0/0  ← the only place that is correct
    ↓
ALB forwards to an ECS task in a PRIVATE subnet
    ↓
Task's security group allows traffic from the ALB's security group only
    ↓
Task queries RDS in the DATA subnet
    ↓
RDS security group allows 5432 from the task's security group only
    ↓
Task pulls a file from S3 → gateway endpoint → never touches NAT, no charge
    ↓
Task calls a third-party payment API → NAT Gateway → internet (billed per GB)
    ↓
Response returns through the ALB
    ↓
─────────────── observability ───────────────
Flow logs record accepted and rejected connections
    ↓
─────────────── a compromise attempt ───────────────
Attacker finds the ALB, exploits the application, lands on the task
    ↓
Tries to scan the VPC → security groups deny everything not explicitly allowed
    ↓
Tries to reach RDS directly → allowed, because the app is allowed
    ↓
Tries to exfiltrate to an unknown host → NAT permits it
    ↓
LESSON: egress restriction and endpoint-only subnets are what limit the blast radius
```

---

## 19. Key Takeaway

> [!IMPORTANT]
> A subnet is public only because its route table says so — put load balancers and NAT in public subnets, everything else in private ones, use security groups that reference each other, and add VPC endpoints to cut both NAT cost and internet exposure.

---

## 20. Common Mistakes

- **Building production in the default VPC**, where everything is public
- **Choosing 10.0.0.0/16** and hitting a CIDR conflict years later
- **Databases in public subnets**, or with `Publicly accessible` enabled
- **Security groups with `0.0.0.0/0`** on administrative ports
- **CIDR-based rules between internal tiers** instead of security-group references
- **No VPC endpoints**, so every S3 and ECR byte is billed through NAT
- **One NAT Gateway shared across AZs**, making it a single point of failure
- **Subnets too small for EKS**, exhausting addresses as pods scale
- **Meshes of VPC peering** where a Transit Gateway was needed
- **Fighting NACLs** without realising they are stateless
- **No flow logs**, leaving no evidence after an incident

---

## 21. Open Source Technologies

- **Terraform / OpenTofu** — the AWS VPC module encodes the layout in this note
- **Pulumi**, **AWS CDK** — the same, in a general-purpose language
- **Prowler**, **ScoutSuite** — audit for open security groups and public exposure
- **Steampipe** — query your network configuration with SQL
- **tcpdump**, **mtr**, **dig** — still the tools that find the actual problem
- **VPC Reachability Analyzer** — AWS's own path tracer; not open source, but it answers "why can't A reach B" faster than reading five configurations

---

## 22. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 23. Workbook Exercise

- [ ] List every security group with a `0.0.0.0/0` inbound rule and justify each one.
- [ ] Check whether any database or cache sits in a subnet with a route to an Internet Gateway.
- [ ] Look at your NAT Gateway data processing charge, then add a gateway endpoint for S3.
- [ ] Draw your VPC from memory, then compare it to reality.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
IGW → public subnets (ALB, NAT) → private subnets (app) → data subnets (DB, no internet)
                                          └── VPC endpoints → S3, ECR, Secrets
```

## 2. Request Flow

```text
Input       a packet, and a route table that decides where it may go
    ↓
Processing  security groups evaluated statefully per interface, NACLs per subnet
    ↓
Output      a connection that either exists or silently does not — flow logs tell you which
```

## 3. Real-World Usage

The three-tier VPC across two availability zones has been the standard production layout for over a decade, and it has held because it maps cleanly onto the failure and trust boundaries that actually matter. Where teams still get hurt is not the topology — it is a CIDR range chosen carelessly, and a NAT Gateway bill nobody looked at until it was the second-largest line item.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A private, software-defined network inside an AWS region |
| **Why does it exist?** | Because cloud instances need the boundary a data centre used to provide |
| **Where does it belong?** | Around everything with a network interface |
| **When should I use it?** | Always — the only question is whether you designed it or accepted the default |
