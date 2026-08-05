# Secrets Management

> **In one line —** keeping API keys, database passwords and signing keys out of your code and out of your git history — where, once committed, they stay forever.

| | |
|---|---|
| **Category** | Security Practice |
| **Architectural Layer** | Infrastructure / Application |
| **Related notes** | [Secure Coding](Secure%20Coding.md) · [OWASP Top 10](OWASP%20Top%2010.md) · [CI CD](../13%20-%20DevOps%20and%20Delivery/CI%20CD.md) · [Docker](../13%20-%20DevOps%20and%20Delivery/Docker.md) · [IAM](../12%20-%20Cloud%20Architecture/IAM.md) |

---

## 1. Short Definition

*What is it?*

Secrets management is how an application receives the credentials it needs — database passwords, API keys, signing keys, certificates — without those values living in source code, container images or configuration files under version control.

---

## 2. Purpose

*What is its main purpose?*

To ensure that reading your source code, pulling your image, or cloning your repository does not hand someone your production credentials.

---

## 3. Problem

*What engineering problem does it solve?*

```text
API_KEY = "sk_live_51H..."   committed to git
    ↓
Cloned by every developer · mirrored by CI · cached by services
    ↓
The repository becomes public, or one laptop is compromised
    ↓
The key is exposed — and deleting the commit changes nothing
```

> [!CAUTION]
> **A secret committed to git is compromised permanently.** History is cloned, forked, cached by GitHub and often mirrored. Rewriting history does not recall those copies. The only correct response is to **rotate the secret**, immediately.
>
> Automated scanners find newly committed keys in public repositories within **minutes**, and exploit them within hours.

---

## 4. Architecture Position

```text
Secrets manager  (Vault, AWS Secrets Manager, Kubernetes Secrets)
    ↓  injected at deploy or fetched at startup
Environment variables / mounted files
    ↓
Application
    ↓
Database · third-party APIs · signing operations
```

---

## 5. The hierarchy of approaches

```text
✗✗ Hard-coded in source
✗  Configuration file committed to git
✗  .env committed to git
~  .env in .gitignore, distributed manually     ← acceptable for local development
✓  Environment variables injected by the platform
✓✓ A secrets manager with rotation and audit logging
✓✓✓ Short-lived, dynamically generated credentials
```

> [!TIP]
> The largest single jump in that list is from "in git" to "not in git". Reach that first, then improve.

---

## 6. Environment variables — the pragmatic default

```bash
# .env — LOCAL ONLY, in .gitignore
DATABASE_URL=postgresql://localhost/dev
STRIPE_KEY=sk_test_...
```

```bash
# .env.example — COMMITTED, values blank
DATABASE_URL=
STRIPE_KEY=
```

> [!CAUTION]
> Environment variables are better than source code but are not strongly protected. They appear in `/proc/<pid>/environ`, in crash dumps, in `docker inspect`, and frequently in logs when a framework prints its configuration on startup. They are a reasonable default, not a strong control.

---

## 7. Secrets managers

| Tool | Notes |
|---|---|
| **HashiCorp Vault** | The most capable; dynamic secrets, leasing, revocation |
| **AWS Secrets Manager / Parameter Store** | Native to AWS, integrates with IAM |
| **Azure Key Vault**, **Google Secret Manager** | The equivalents |
| **Kubernetes Secrets** | Convenient, but **base64 is not encryption** — enable encryption at rest and RBAC |
| **SOPS**, **sealed-secrets** | Encrypted secrets *can* be committed to git safely |

**What a real secrets manager adds beyond environment variables:**

```text
Centralised storage with encryption at rest
Fine-grained access control per service
AUDIT LOG — who read which secret, and when
Automatic ROTATION
Dynamic secrets — a database credential created per session, valid for an hour
```

---

## 8. Rotation

> [!IMPORTANT]
> **A secret you cannot rotate quickly is a secret you cannot respond to.** The question is not whether a credential will ever leak, but how long the exposure lasts once it does.

```text
Static secret, manually rotated     → exposure lasts until someone notices
Automated rotation, 30 days         → bounded
Dynamic, per-session, 1 hour        → nearly self-healing
```

Design for rotation from the start: applications must re-read credentials rather than caching them at boot, and two credentials must be valid simultaneously during a rollover.

---

## 9. Real World Example

- **Uber (2016)** — AWS credentials found in a private GitHub repository led to a large breach. Private is not the same as safe.
- **Codecov (2021)** — a compromised CI script harvested environment variables from thousands of build pipelines.
- **Public repository scanning** is routine and automated; keys committed to a public repo are typically exploited the same day.

---

## 10. Secrets in containers and CI

```text
✗ ENV API_KEY=... in a Dockerfile      — baked into an image layer, forever
✗ Secrets in build arguments           — visible in image history
✓ Injected at RUNTIME by the orchestrator
✓ Mounted as files rather than environment variables where possible

CI/CD:
✓ Use the platform's encrypted secret store
✓ Mask secrets in logs — verify this actually works
✗ Never echo a secret, even to debug
```

> [!CAUTION]
> **Docker image layers are permanent.** A secret added in one layer and deleted in a later one is still present in the image and extractable with `docker history`.

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Use environment variables as a baseline for every project from day one. Move to a secrets manager once you have production systems, more than one environment, or any compliance requirement.

> [!CAUTION]
> Do not deploy Vault for a hobby project — the operational cost is real, and an unavailable secrets manager means an application that cannot start. Match the tool to the actual risk.

---

## 12. Advantages and Disadvantages

**Advantages of a secrets manager**
- Central control and revocation
- Audit trail of every access
- Automated rotation
- Dynamic, short-lived credentials
- Per-service access policies

**Disadvantages**
- Another critical dependency at startup
- Operational complexity and its own access control to manage
- Cost, for managed services
- Bootstrapping problem: how does the application authenticate to fetch its secrets?

> [!TIP]
> The bootstrapping problem is solved by **workload identity** — the cloud platform attests that this instance or pod is who it claims to be, so no static credential is needed to obtain the others. See [IAM](../12%20-%20Cloud%20Architecture/IAM.md).

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Startup** | One fetch, cached in memory |
| **Rotation** | Requires re-reading; do not cache forever |
| **Availability** | The secrets manager becomes a startup dependency |
| **Dynamic secrets** | A credential request per session — plan for the load |

---

## 14. Security Considerations

- **Secret scanning in CI** — gitleaks or trufflehog, blocking the commit rather than reporting after the fact
- **Least privilege per secret** — a service reads only what it needs
- **Different secrets per environment** — never reuse production credentials in staging
- **Never log configuration on startup** — frameworks that print their settings are a common leak path
- **Rotate on staff departure**, not only on suspicion of compromise
- **Encrypt Kubernetes Secrets at rest** and restrict RBAC — base64 encoding provides no protection whatsoever
- **Treat backups as secret-bearing** — a database dump contains everything the database contained

---

## 15. Mental Model

> [!NOTE]
> **Secrets are house keys, and git is a photocopier that never forgets.**
>
> Once a key has been through it, every copy that was ever made still opens the door. You cannot un-photocopy it. The only real response is to change the lock — which is why rotation, not deletion, is the answer to a leaked secret.

---

## 16. Mini Architecture Diagram

```text
Developer machine        .env (gitignored)
        ↓
CI/CD                    encrypted secret store, masked in logs
        ↓
Orchestrator             injects at runtime, never baked into the image
        ↓
Application              reads at startup, re-reads on rotation
        ↓
Secrets manager          audit log · rotation · per-service policy
```

---

## 17. Complete Request Flow

```text
Deployment starts
    ↓
Pod starts with a WORKLOAD IDENTITY — no static credential
    ↓
Authenticates to the secrets manager using that identity
    ↓
Fetches only the secrets its policy allows
    ↓
Access recorded in the audit log
    ↓
Secrets held in memory, never written to disk or logged
    ↓
Application connects to the database
    ↓
─────────── rotation ───────────
New credential created; both old and new valid briefly
    ↓
Application re-reads and reconnects
    ↓
Old credential revoked
    ↓
─────────── leak response ───────────
A key appears in a public repository
    ↓
ROTATE FIRST, investigate second
    ↓
Audit log shows whether it was used, by whom, and when
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> Never commit a secret — and if one is committed, rotate it rather than deleting the commit, because history is permanent and already copied.

---

## 19. Common Mistakes

- **Secrets in source code or in committed `.env` files**
- **Deleting a leaked commit instead of rotating the key**
- **`ENV SECRET=...` in a Dockerfile**, baking it into the image
- **Treating a private repository as safe storage**
- **The same credentials across staging and production**
- **Base64 in Kubernetes Secrets** mistaken for encryption
- **Frameworks logging configuration on startup**
- **No rotation plan** until an incident forces one

---

## 20. Open Source Technologies

- **HashiCorp Vault**, **OpenBao** — full secrets management with dynamic credentials
- **SOPS**, **sealed-secrets**, **age** — encrypted secrets safe to commit
- **External Secrets Operator** — sync a cloud manager into Kubernetes
- **gitleaks**, **trufflehog** — pre-commit and CI scanning
- **direnv**, **python-dotenv** — local development ergonomics
- **cloud-native**: AWS Secrets Manager, Azure Key Vault, Google Secret Manager

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Run gitleaks over your repository history and see what it finds.
- [ ] Check whether any secret is present in a Docker image layer (`docker history`).
- [ ] Write down how you would rotate your database password today, and how long it would take.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Secrets manager → runtime injection → application memory → external services
```

## 2. Request Flow

```text
Input       a workload identity proving which service is asking
    ↓
Processing  policy-checked retrieval, recorded in an audit log
    ↓
Output      credentials in memory only, rotatable without redeployment
```

## 3. Real-World Usage

**Automated scanners find keys committed to public GitHub repositories within minutes.** This is why every serious CI pipeline now includes secret scanning that blocks the push rather than reporting afterwards.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | How credentials reach an application without living in code or git |
| **Why does it exist?** | Because a committed secret is permanently exposed |
| **Where does it belong?** | Between your infrastructure and your running application |
| **When should I use it?** | From the first commit — with rotation planned before it is needed |
