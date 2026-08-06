# Cloud Security

> **In one line —** protecting systems where the perimeter no longer exists; in practice, a small number of mistakes cause almost all breaches, and they are all preventable configuration.

| | |
|---|---|
| **Category** | Concept / Practice |
| **Architectural Layer** | Security |
| **Related notes** | [IAM](IAM.md) · [VPC](VPC.md) · [Cloud Fundamentals](Cloud%20Fundamentals.md) · [Secrets Management](../09%20-%20Security/Secrets%20Management.md) · [Security Fundamentals](../09%20-%20Security/Security%20Fundamentals.md) · [Encryption](../09%20-%20Security/Encryption.md) |

---

## 1. Short Definition

*What is it?*

Cloud security is the practice of securing systems where **you do not control the hardware, the network is software, and every resource is reachable through a public API.** It is less about firewalls than about identity, configuration and evidence.

---

## 2. Problem

*What engineering problem does it solve?*

```text
The old model
    a building, a firewall, a network you owned
    "inside" was meaningful
        ↓
The cloud model
    the control plane is an internet-facing API
    a credential works from anywhere on earth
    resources are created and destroyed hourly by automation
        ↓
Every assumption the old model rested on is gone
```

> [!IMPORTANT]
> **The industry's uncomfortable finding is that cloud breaches are almost never sophisticated.** They are a public storage bucket, an over-permissive role, a key committed to a repository, or a database open to the internet. The provider's infrastructure is not what fails. **Misconfiguration is the threat model** — which is good news, because configuration is something you can review, test and enforce.

---

## 3. Architecture Position

```text
              ┌──── GOVERNANCE ────┐
              │ Organizations, SCPs │  what accounts may do at all
              └──────────┬──────────┘
                         │
              ┌──────────▼──────────┐
              │      IDENTITY        │  ← the real perimeter
              │  IAM, SSO, roles     │
              └──────────┬──────────┘
                         │
     ┌───────────────────┼───────────────────┐
     ▼                   ▼                   ▼
  NETWORK             DATA               WORKLOAD
  VPC, SGs        encryption, KMS     images, patching,
  endpoints       backups, DLP        dependencies
     └───────────────────┼───────────────────┘
                         ▼
              ┌─────────────────────┐
              │     DETECTION        │  CloudTrail, GuardDuty,
              │     and RESPONSE     │  Config, alarms
              └─────────────────────┘
```

---

## 4. The shared responsibility model, precisely

```text
AWS SECURES                          YOU SECURE
physical data centres                 who can call the API
hypervisor and host OS                network rules you define
managed service internals             which data is encrypted, and with whose key
the network backbone                  OS patching on your instances
service availability                  your application code and dependencies
                                      your data, and who can read it
```

> [!CAUTION]
> **Higher-level services move the line but never remove your half.** On S3 the provider handles durability and encryption — you still decide whether the bucket is public. On Lambda they patch the runtime — you still own your dependencies. There is no service where "managed" means "secured", and reading the responsibility split for each service you adopt is genuinely part of the work.

---

## 5. The mistakes that cause real breaches

```text
1. PUBLIC STORAGE            a bucket exposed to the internet
2. OVER-PERMISSIVE IAM       Action:* Resource:* on something trivial
3. LEAKED CREDENTIALS        a long-lived key in git, CI logs, or a laptop
4. OPEN SECURITY GROUPS      0.0.0.0/0 on 22, 3389, 5432, 6379, 9200
5. UNPATCHED WORKLOADS       an instance or image nobody has rebuilt in a year
6. NO MFA                    on the root account or an administrator
7. DISABLED OR LOCAL LOGGING an intruder deletes the evidence
8. PUBLIC MANAGED DATABASES  "Publicly accessible: yes"
```

> [!IMPORTANT]
> **That list is close to exhaustive for real-world incidents, and every item is a configuration check.** You do not need a threat-intelligence programme to prevent them — you need Block Public Access enabled, no long-lived keys, security groups that reference security groups, and automated detection that tells you when one of these appears. Fix these eight and you are ahead of most organisations.

---

## 6. Defence in depth, applied

```text
GUARDRAILS      SCPs deny whole classes of action, account-wide
                → cannot disable CloudTrail, cannot leave approved regions

IDENTITY        roles not keys, SSO with MFA, least privilege, boundaries

NETWORK         private subnets, SG-to-SG rules, endpoints instead of NAT
                → egress restriction, because exfiltration goes outbound

DATA            encryption everywhere, KMS key policies as a second gate,
                versioning and Object Lock so backups survive an attacker

WORKLOAD        minimal images, scanned dependencies, no SSH, IMDSv2

DETECTION       CloudTrail to a separate account, GuardDuty, Config rules
                → alarms that a human actually receives
```

> [!TIP]
> **Service Control Policies are the most underused control in AWS.** They apply above IAM, so even an account administrator cannot bypass them. A handful — deny disabling CloudTrail, deny deleting log buckets, deny unapproved regions, deny root usage — converts your most dangerous configuration mistakes into impossibilities. They cost nothing.

---

## 7. Account separation as a security boundary

```text
ONE ACCOUNT
    every permission mistake is a production permission mistake

MULTI-ACCOUNT
    management     billing and guardrails only
    production     locked down, few humans, changes through CI
    staging        realistic, safe to break
    development    per-team, capped spend
    log archive    write-only; nobody can delete from it
    security       read-only access into everything, for the security team
```

> [!IMPORTANT]
> **An account boundary is stronger than any policy, because it requires no policy to be correct.** A development role cannot accidentally drop a production table that exists in a different account — not because a rule forbids it, but because there is no path. This is the single most effective structural decision available, and it is dramatically easier to do early.

---

## 8. Encryption and key management

```text
AT REST      enable it everywhere; it is free or nearly free
             SSE-S3 / default EBS encryption → no reason not to

IN TRANSIT   TLS everywhere, including inside your VPC
             deny non-TLS requests by policy condition

KMS          customer-managed keys give you a SECOND authorisation layer
             a key policy can deny access even when IAM allows it
             → and it produces an audit trail of every decrypt
```

> [!TIP]
> **A KMS key policy is a second lock on the same door, and it fails independently of IAM.** Encrypting sensitive data with a customer-managed key means an over-permissive IAM policy alone is not sufficient to read it — the key policy must also allow it. This is also the layer people forget when cross-account access mysteriously fails.

---

## 9. Real World Example

- **The 2019 Capital One breach** — an SSRF vulnerability used to reach the instance metadata service and steal role credentials. IMDSv2 and a tighter role scope both would have contained it.
- **Repeated public-bucket incidents** across many industries — always Block Public Access being off.
- **Keys in public GitHub repositories** — scanned by attackers within minutes, typically used for cryptomining.
- **Exposed Redis, Elasticsearch and MongoDB** — internal services deployed with no authentication on a reachable network.
- **Supply chain compromise** — a malicious dependency in a build with permissions to deploy.

---

## 10. Communication and Dependencies

- **[IAM](IAM.md)** — the foundation; nothing else compensates for getting it wrong
- **AWS Organizations and SCPs** — the guardrail layer above IAM
- **CloudTrail in a separate account** — evidence you can still trust after a compromise
- **GuardDuty, Config, Security Hub** — detection that costs little relative to its value
- **Secrets Manager or Parameter Store** — see [Secrets Management](../09%20-%20Security/Secrets%20Management.md)
- **A CI pipeline** that scans infrastructure code and dependencies before merge

---

## 11. When To Use / When NOT To Use

> [!TIP]
> All of this applies from the first day of an account, and the cheap parts — Block Public Access, MFA, CloudTrail, default encryption, no long-lived keys — take under an hour. Do those before the first production workload, because retrofitting them later means touching everything.

> [!CAUTION]
> - **Do not buy tools before fixing configuration** — a scanner that reports the same eight findings for two years has not improved anything
> - **Do not treat compliance as security** — passing an audit and being secure overlap only partly
> - **Do not build a bastion host** by reflex; SSM Session Manager is logged and needs no open port
> - **Do not encrypt everything with customer-managed keys** indiscriminately — you inherit key management, rotation and cross-account complexity
> - **Do not add alerts nobody reads** — an ignored alarm is worse than none, because it manufactures false confidence

---

## 12. Advantages and Disadvantages

**Advantages of the cloud model**
- Configuration is code, so security becomes reviewable in a pull request
- Encryption and audit logging are one flag away
- Identity-based controls are far finer than network controls ever were
- Account boundaries provide isolation no on-premises network matched
- Detection services are cheap relative to building equivalents
- Patching disappears entirely for managed services

**Disadvantages**
- **One credential can reach everything, from anywhere**
- Misconfiguration is instant, silent and global
- The number of controls is large and their interaction is subtle
- Defaults optimise for making things work, not for safety
- Visibility below your layer is limited — you trust the provider's assertions
- Automation creates and destroys resources faster than humans can review
- Shared responsibility is genuinely misunderstood by most teams

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Encryption at rest** | Effectively zero; hardware accelerated |
| **TLS in transit** | Negligible on modern CPUs |
| **KMS** | An API call per data key, with request quotas — cache data keys |
| **VPC endpoints** | Faster and cheaper than routing through NAT |
| **CloudTrail data events** | No latency cost, but real storage cost at high volume |
| **GuardDuty, Config** | Out of band; no effect on your request path |

> [!TIP]
> **Security controls in the cloud are cheap in performance and occasionally expensive in cost.** Nobody should skip encryption for speed. The item worth watching is log volume: CloudTrail data events on a busy bucket and verbose CloudWatch ingestion can each become a significant bill, so scope them to what you would actually investigate.

---

## 14. Security Considerations

> [!CAUTION]
> **Assume a credential will leak and design so that it does not matter much.** Short-lived credentials, tight scopes, account separation, and immutable audit logs in an account the credential cannot reach. Prevention will eventually fail somewhere; what determines whether it becomes an incident or a breach is blast radius and evidence.

- **Root account** — hardware MFA, no keys, effectively unused
- **No long-lived access keys anywhere** — roles and OIDC cover nearly every case
- **Block Public Access at the account level** on day one
- **CloudTrail to a write-only bucket in a separate account**, with Object Lock
- **SCP guardrails** for logging, regions and root usage
- **Secret scanning in CI**, and rotate anything that ever appeared in a repository
- **IMDSv2 required** on all instances; see [EC2](EC2.md)
- **Egress restriction** — exfiltration is outbound, and unrestricted NAT is the path
- **An incident plan that includes revoking sessions**, not just disabling keys

---

## 15. Mental Model

> [!NOTE]
> **Old security was a castle: walls, a gate, and trust for anyone inside. Cloud security is an airport.**
>
> There is no inside — every door checks your pass, every pass expires, and every check is recorded. Nobody argues that being past the first checkpoint means anything at the next one. The failures at an airport are not breached walls; they are a door propped open, a pass issued too broadly, or a camera nobody was watching. That is exactly the shape of real cloud incidents.

---

## 16. Mini Architecture Diagram

```text
┌──────────── AWS ORGANIZATION ─────────────────────────────┐
│  SCPs: no disabling CloudTrail · no unapproved regions     │
│                                                            │
│  ┌── management ──┐  ┌── log archive ─────────────────┐    │
│  │ billing, SCPs  │  │ write-only, Object Lock, no    │    │
│  │ nothing runs   │  │ delete permission for anyone   │    │
│  └────────────────┘  └────────────▲───────────────────┘    │
│                                    │ CloudTrail            │
│  ┌── PRODUCTION ───────────────────┼──────────────────┐    │
│  │  SSO + MFA → short-lived roles  │                  │    │
│  │  ┌── VPC ──────────────────┐    │                  │    │
│  │  │ public: ALB only        │    │                  │    │
│  │  │ private: app (no SSH)   │    │                  │    │
│  │  │ data: RDS, no internet  │    │                  │    │
│  │  │ endpoints, not NAT      │    │                  │    │
│  │  └─────────────────────────┘    │                  │    │
│  │  KMS keys · encrypted at rest   │                  │    │
│  │  GuardDuty · Config · Security Hub                 │    │
│  └────────────────────────────────────────────────────┘    │
│                                                            │
│  ┌── staging ──┐  ┌── dev ──┐  ┌── security (read-only) ┐  │
│  └─────────────┘  └─────────┘  └────────────────────────┘  │
└────────────────────────────────────────────────────────────┘
```

---

## 17. Complete Request Flow

```text
An engineer needs to change production
    ↓
Logs in via SSO with MFA → assumes a role for 1 hour
    ↓
Opens a pull request; CI scans the Terraform for public exposure
and open security groups → a finding blocks the merge
    ↓
Fixed, reviewed, merged; CI assumes a deploy role via OIDC (no stored key)
    ↓
Change applied; every API call recorded in the log archive account
    ↓
─────────────── an attack ───────────────
An SSRF bug is found in the public application
    ↓
Attacker tries the metadata service → IMDSv2 blocks the naive request
    ↓
Attacker instead uses the application's own database access
    ↓
Tries to reach the billing bucket → IAM denies; the attempt is logged
    ↓
Tries to exfiltrate to an unknown host → egress rules block it
    ↓
GuardDuty flags anomalous behaviour → alarm → on-call paged
    ↓
Session revoked, task terminated, the ASG replaces it from a clean image
    ↓
CloudTrail in the log archive account shows exactly what was touched —
and the attacker never had permission to delete that trail
    ↓
─────────────── the conclusion ───────────────
The vulnerability was real. The breach was not,
because the blast radius was bounded and the evidence survived.
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> Cloud breaches are misconfiguration, not sophistication — fix the eight common mistakes, separate production into its own account, use short-lived credentials, and keep an audit trail an attacker cannot delete.

---

## 19. Common Mistakes

- **Believing the provider secures your data** — they secure their infrastructure
- **One account for everything**, so every mistake is a production mistake
- **Long-lived access keys**, which eventually reach a repository or a laptop backup
- **Public buckets** because Block Public Access was never enabled
- **`0.0.0.0/0`** on administrative or database ports
- **CloudTrail in the same account it audits**, deletable by whoever gets in
- **No MFA on root or administrators**
- **Alerts routed to a channel nobody reads**
- **Buying tools instead of fixing configuration**
- **Treating a passed audit as evidence of security**
- **No egress restriction**, so exfiltration is trivial once inside
- **No rehearsed response** — nobody knows how to revoke a session under pressure

---

## 20. Open Source Technologies

- **Prowler**, **ScoutSuite**, **CloudSploit** — audit an account against known misconfigurations
- **cloudsplaining**, **Parliament**, **PMapper** — analyse IAM for over-permission and escalation paths
- **Checkov**, **tfsec**, **Terrascan** — scan infrastructure code in CI before it is applied
- **trivy**, **grype** — scan container images and dependencies
- **gitleaks**, **trufflehog** — find secrets in repositories and history
- **Falco** — runtime detection for containers
- **OPA / Conftest** — enforce policy on infrastructure changes
- **CloudQuery**, **Steampipe** — query your whole estate as a database

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Run Prowler against one account and read only the high findings.
- [ ] Walk the eight-item list in section 5 and mark each one pass or fail, honestly.
- [ ] Verify CloudTrail is enabled, in all regions, and delivered somewhere you cannot delete.
- [ ] Write down what you would do in the first ten minutes if a key leaked. If you cannot, that is the gap.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
SCPs → identity (SSO, roles) → network + data + workload controls → detection
                                                                  └→ immutable audit trail
```

## 2. Request Flow

```text
Input       a credential, and a configuration someone wrote
    ↓
Processing  guardrails, then identity, then network and data controls, all recorded
    ↓
Output      an action allowed or denied — and evidence that survives the actor
```

## 3. Real-World Usage

Every public post-mortem of a cloud breach reads similarly: a known misconfiguration, an over-broad credential, and logs that were either missing or deletable. The organisations that handle incidents well are rarely the ones with the most tools — they are the ones whose blast radius was small and whose evidence was intact.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Securing systems where identity, not the network, is the boundary |
| **Why does it exist?** | Because the control plane is a public API and misconfiguration is instant |
| **Where does it belong?** | Every layer, starting with account structure and identity |
| **When should I use it?** | From the first hour of a new account — retrofitting costs far more |
