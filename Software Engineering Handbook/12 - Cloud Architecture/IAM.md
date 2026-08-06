# IAM

> **In one line —** the service that decides whether every single AWS API call is allowed; learn it first, because in the cloud identity is the perimeter.

| | |
|---|---|
| **Full name** | AWS Identity and Access Management |
| **Category** | Identity and Access Control |
| **Architectural Layer** | Security |
| **Related notes** | [AWS Architecture](AWS%20Architecture.md) · [Cloud Security](Cloud%20Security.md) · [Authentication](../09%20-%20Security/Authentication.md) · [Authorization](../09%20-%20Security/Authorization.md) · [Secrets Management](../09%20-%20Security/Secrets%20Management.md) · [EC2](EC2.md) |

---

## 1. Short Definition

*What is it?*

IAM answers one question, several billion times a day: **is this principal allowed to perform this action on this resource under these conditions?** Every AWS call — from launching a thousand servers to reading one object — passes through it.

---

## 2. Problem

*What engineering problem does it solve?*

```text
A traditional data centre
    ↓
Physical access + a network perimeter + a firewall
    ↓
Being inside the building meant something
    ↓
─────────────────────────────────────
The cloud
    ↓
The control plane is a public API
    ↓
A credential is the ONLY thing standing between the internet
and the ability to delete your entire company's infrastructure
```

> [!IMPORTANT]
> **This is the mental shift that matters more than any other in cloud engineering: identity replaced the network as the security boundary.** There is no perimeter to be inside of. A leaked key works equally well from a laptop in your office and a server on the other side of the world. Everything else in cloud security is a second line of defence behind this one.

---

## 3. Architecture Position

```text
Any caller  →  console, CLI, SDK, or a service acting on your behalf
                            │
                    ┌───────▼────────┐
                    │      IAM       │  evaluates policies
                    └───────┬────────┘
                     allow  │  deny
                            ▼
            S3 · EC2 · RDS · Lambda · every other service
                            │
                    CloudTrail records the call either way
```

IAM sits in front of everything. It is not a component you integrate with — it is the gate every other note in this folder passes through.

---

## 4. The core objects

```text
USER          a long-lived identity for a human      → avoid; use SSO instead
GROUP         a bundle of users sharing policies
ROLE          an identity anything can ASSUME temporarily   ← the important one
POLICY        a JSON document listing allowed/denied actions
PRINCIPAL     whoever is making the call
```

```text
ROLE = permissions + a trust policy saying WHO may assume it
    ↓
assume it → temporary credentials (15 min to 12 h) → they expire on their own
```

> [!IMPORTANT]
> **Roles are the answer to almost every IAM question, because credentials that expire cannot be leaked in a lasting way.** An EC2 instance gets a role, not keys. A Lambda function gets a role. A CI pipeline gets a role via OIDC. A human gets a role via SSO. If your design involves an access key in a file, an environment variable or a secrets manager, there is nearly always a role-based alternative — and it is strictly better.

---

## 5. Anatomy of a policy

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "ReadOwnTenantUploads",
    "Effect": "Allow",
    "Action": ["s3:GetObject"],
    "Resource": "arn:aws:s3:::uploads/tenant-a/*",
    "Condition": {
      "Bool": { "aws:SecureTransport": "true" }
    }
  }]
}
```

```text
Effect      Allow or Deny
Action      the API calls  (s3:GetObject, ec2:*)
Resource    which ARNs — this is where least privilege lives
Condition   the underused part: source IP, MFA, tags, TLS, time
Principal   in resource-based policies only — who is being granted access
```

> [!TIP]
> **Conditions are where IAM becomes genuinely expressive, and almost nobody uses them.** `aws:PrincipalTag` for attribute-based access, `aws:MultiFactorAuthPresent` to require MFA for destructive actions, `aws:SourceVpce` to allow a bucket to be read only through your VPC endpoint. These turn a coarse permission into a precise one without writing more policies.

---

## 6. How a decision is actually made

```text
1. Is there an explicit DENY anywhere?          → DENIED. Nothing overrides this.
2. Does an SCP (organisation policy) allow it?  → if not, DENIED
3. Does a permission boundary allow it?         → if not, DENIED
4. Is there an explicit ALLOW?                  → ALLOWED
5. Otherwise                                    → DENIED (default)
```

> [!IMPORTANT]
> **Two rules explain every confusing IAM error.** First, **the default is deny** — nothing is permitted until something says so. Second, **an explicit deny always wins**, at any layer, regardless of how many allows exist elsewhere. When a call fails despite an obviously correct policy, you are looking for a deny you did not write — usually in a Service Control Policy, a permission boundary or a resource policy.

---

## 7. Identity policies versus resource policies

```text
IDENTITY-BASED       attached to a user or role   "this role may read that bucket"
RESOURCE-BASED       attached to the resource     "this bucket may be read by that role"

SAME ACCOUNT       → EITHER one allowing it is enough
CROSS-ACCOUNT      → BOTH must allow it
```

> [!CAUTION]
> **Cross-account access requires permission on both sides, and this catches everyone once.** The role's policy must allow the action *and* the target resource's policy must allow the role. When a cross-account S3 read fails with access denied despite a correct role policy, the bucket policy is what is missing — or the KMS key policy, which is the same trap one layer deeper.

---

## 8. Least privilege, practically

```text
HOW IT ACTUALLY GETS DONE
    1. start with nothing
    2. grant the specific actions the code needs
    3. run it; read the CloudTrail access-denied entries
    4. add exactly what was missing
    5. use IAM Access Analyzer to generate a policy from real usage
    6. revisit and remove unused permissions periodically
```

```text
WARNING SIGNS
    "Action": "*"                    → an administrator, whatever it is called
    "Resource": "*"                  → every object in the account
    AdministratorAccess on a service → nobody scoped it, ever
    iam:PassRole with Resource "*"   → PRIVILEGE ESCALATION
```

> [!CAUTION]
> **`iam:PassRole` and `iam:CreatePolicyVersion` are permission-escalation primitives, and they look harmless in a policy review.** If a principal can pass any role to a service, it can launch a function or instance carrying the administrator role and inherit those permissions. Several innocuous-looking permissions have this property. Scope `PassRole` to specific roles, always.

---

## 9. Real World Example

- **An EC2 instance role** reading configuration from S3 and secrets from Secrets Manager, with no keys on disk; see [EC2](EC2.md).
- **A Lambda execution role** per function, each scoped to its own queue and table; see [Lambda](Lambda.md).
- **GitHub Actions assuming a role via OIDC**, which removes long-lived deployment keys entirely; see [GitHub Actions](../13%20-%20DevOps%20and%20Delivery/GitHub%20Actions.md).
- **Engineers via IAM Identity Center (SSO)**, with short-lived sessions and no IAM users at all.
- **Cross-account roles** — a monitoring account assuming a read-only role in production.
- **A tenant-scoped policy** using tags so one role serves many tenants safely.

---

## 10. Communication and Dependencies

- **CloudTrail** — the record of every decision; without it you are blind
- **AWS Organizations and SCPs** — account-wide guardrails above IAM
- **IAM Identity Center** — how humans should get access
- **KMS key policies** — a separate authorisation layer people forget
- **OIDC providers** — for CI systems and Kubernetes service accounts
- **IAM Access Analyzer** — finds external access and generates least-privilege policies

---

## 11. When To Use / When NOT To Use

> [!TIP]
> IAM is not optional and has no alternative on AWS. The real decisions are: roles instead of users, SSO instead of long-lived credentials, and permissions scoped to resources instead of `*`.

> [!CAUTION]
> - **Do not create IAM users for humans** — use IAM Identity Center
> - **Do not create long-lived access keys** for anything that can assume a role
> - **Do not use IAM for your application's own user authorisation** — it governs AWS API calls, not your product's permissions
> - **Do not hand-write a policy per resource** — use tags and conditions
> - **Do not rely on IAM alone** for tenant isolation of your customers' data
> - **Do not use the root account** for anything routine

---

## 12. Advantages and Disadvantages

**Advantages**
- Extremely fine-grained — every API action is individually controllable
- Temporary credentials by default when roles are used
- Conditions allow context-aware rules (MFA, IP, tags, TLS)
- Cross-account access without shared secrets
- Every decision auditable through CloudTrail
- Free
- Access Analyzer can generate policies from real activity

**Disadvantages**
- **Genuinely hard to learn** — several policy types and an evaluation order
- Error messages are often unhelpfully vague
- Easy to grant too much and hard to notice afterwards
- Policy sprawl becomes unmanageable without conventions
- Some escalation paths are non-obvious (`PassRole`, policy versions)
- Eventually consistent — a new policy may take moments to apply
- Character limits push teams toward broader wildcards

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Authorisation** | Sub-millisecond, on every call, invisible to you |
| **Role assumption** | One STS call; cache the credentials and reuse them |
| **Credential expiry** | Handled by the SDKs — do not implement refresh yourself |
| **Policy propagation** | Eventually consistent; seconds after a change |
| **Policy size limits** | Real constraints that shape how you organise permissions |

> [!TIP]
> **Let the SDK's credential provider chain do its job.** It finds the instance role, assumes what is needed, caches the result and refreshes before expiry. Almost every "credentials expired mid-job" incident comes from code that fetched a token once and held it. Do not reinvent this.

---

## 14. Security Considerations

> [!CAUTION]
> **The root account is the one identity that cannot be constrained by any policy.** Enable MFA on it with a hardware key, delete its access keys, use a mail alias several people monitor, and then do not use it. Almost every catastrophic AWS incident involves either the root account or a long-lived access key that reached a public repository.

- **No long-lived access keys** — roles and OIDC eliminate nearly every use case
- **MFA everywhere for humans**, and required by condition for destructive actions
- **SCPs as guardrails** — deny leaving regions, deny disabling CloudTrail, deny root usage
- **CloudTrail in every region**, delivered to a separate account an attacker cannot reach
- **Permission boundaries** so teams can create roles without escalating themselves
- **Rotate and review** — Access Analyzer and credential reports show what is unused
- **GuardDuty** detects anomalous credential use, including keys used from unexpected locations
- **Assume a leaked key exists somewhere** and design so that its blast radius is small

---

## 15. Mental Model

> [!NOTE]
> **IAM is a bouncer with an extremely literal rulebook.**
>
> Nobody enters unless a rule says so — and if any rule anywhere says "not this person", no other rule can overrule it. The bouncer does not care whether you are inside the building; there is no inside. What you carry is a wristband that dissolves after an hour, which is why stealing one is far less useful than stealing a key. And there is one master key holder — the root account — who obeys nobody, which is why you keep them at home.

---

## 16. Mini Architecture Diagram

```text
        HUMANS                          MACHINES
   IAM Identity Center            EC2 / ECS / Lambda
   (SSO, MFA, short sessions)     (attached role)
            │                              │
            │ assume role                  │ credentials from
            │ (1 hour)                     │ the metadata service
            ▼                              ▼
   ┌──────────────────── IAM EVALUATION ────────────────────┐
   │  explicit Deny?  →  SCP?  →  boundary?  →  Allow?      │
   └────────────────────────┬───────────────────────────────┘
                            │
    ┌───────────────────────┼───────────────────────┐
    ▼                       ▼                       ▼
   S3 bucket policy    KMS key policy         every other service
   (must ALSO allow    (a separate layer
    cross-account)      people forget)
                            │
                    CloudTrail → log archive account (write-only)

   CI/CD:  GitHub Actions ──OIDC──► role   (no stored keys at all)
```

---

## 17. Complete Request Flow

```text
An engineer logs in through IAM Identity Center with MFA
    ↓
Assumes the "developer" role in the staging account — a 1-hour session
    ↓
Runs a CLI command; the SDK signs it with the temporary credentials
    ↓
IAM evaluates: no explicit deny → SCP allows → boundary allows → policy allows
    ↓
The call succeeds; CloudTrail records who, what, when and from where
    ↓
─────────────── the same, for a machine ───────────────
An ECS task starts with a task role attached
    ↓
The SDK fetches temporary credentials from the container credentials endpoint
    ↓
Reads its database password from Secrets Manager — allowed, scoped to one secret
    ↓
Writes to s3://uploads/tenant-a/* — the condition on the tenant tag matches
    ↓
Attempts to read s3://billing-exports/ → DENIED, and logged
    ↓
Credentials refresh automatically every few hours; nothing is stored anywhere
    ↓
─────────────── a leak ───────────────
An old access key appears in a public repository
    ↓
GuardDuty flags anomalous use from an unfamiliar location
    ↓
The key is disabled; CloudTrail shows exactly what it touched
    ↓
Blast radius was one bucket prefix, because it was never granted more
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> Identity is the perimeter — use roles and SSO so credentials expire on their own, scope every policy to specific actions and resources, remember that an explicit deny always wins, and keep CloudTrail somewhere an intruder cannot delete it.

---

## 19. Common Mistakes

- **Long-lived access keys** in code, CI, or a laptop's config file
- **IAM users for humans** instead of Identity Center
- **`Action: "*"` and `Resource: "*"`** because scoping felt slow
- **The root account used routinely**, without MFA
- **`iam:PassRole` unscoped**, quietly granting privilege escalation
- **Forgetting the resource policy** in cross-account access, then blaming the role
- **Forgetting the KMS key policy**, which denies the call one layer below S3
- **No CloudTrail**, or CloudTrail written to the same account it audits
- **Never removing unused permissions**, so policies only ever grow
- **No SCP guardrails**, so any account can disable logging
- **Using IAM as your application's authorisation model** for end users

---

## 20. Open Source Technologies

- **aws-vault**, **Leapp**, **granted** — assume roles without keys on disk
- **Prowler**, **ScoutSuite**, **cloudsplaining** — audit policies for over-permission
- **PMapper**, **IAM Vulnerable** — find privilege-escalation paths in your own account
- **Terraform / OpenTofu** — policies as reviewed code
- **Steampipe** — query IAM configuration with SQL
- **OPA / Conftest** — validate policy documents in CI before they are applied
- **Parliament** — lint IAM policies for errors and bad patterns

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Download a credential report and list every long-lived access key. Delete the ones you can.
- [ ] Find every policy containing `"Action": "*"` and write down who or what holds it.
- [ ] Check that the root account has MFA and no access keys.
- [ ] Search for `iam:PassRole` with an unscoped resource.
- [ ] Replace one stored CI credential with an OIDC role.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
principal (role, short-lived) → IAM evaluation (deny > SCP > boundary > allow) → service
                                                    └── CloudTrail → separate account
```

## 2. Request Flow

```text
Input       a signed API call carrying temporary credentials
    ↓
Processing  explicit deny checked first, then organisation policy, boundary, then allow
    ↓
Output      allowed or denied — and recorded either way
```

## 3. Real-World Usage

Every serious AWS estate converges on the same pattern: humans through SSO, machines through attached roles, CI through OIDC, guardrails through SCPs, and no long-lived keys anywhere. The reason is not elegance — it is that the alternative has produced most of the industry's largest cloud breaches, usually starting with a key in a repository.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | The authorisation layer in front of every AWS API call |
| **Why does it exist?** | Because in the cloud a credential is the only boundary that remains |
| **Where does it belong?** | In front of everything — it is not optional and has no bypass |
| **When should I use it?** | Always, and deliberately: roles over keys, scoped over wildcard |
