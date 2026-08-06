# Security Checklist

> **In one line —** the small number of things that cause most real breaches; work down this list before buying a tool, because almost every item is configuration you already control.

| | |
|---|---|
| **Category** | Template |
| **Use for** | A new service, a new cloud account, a security review, or an audit response |
| **Related notes** | [Secure Coding](../09%20-%20Security/Secure%20Coding.md) · [Cloud Security](../12%20-%20Cloud%20Architecture/Cloud%20Security.md) · [IAM](../12%20-%20Cloud%20Architecture/IAM.md) · [Code Review Checklist](Code%20Review%20Checklist.md) · [Login System Architecture](../15%20-%20Real%20World%20System%20Design/Login%20System%20Architecture.md) |

---

## Start here: the ten that cause real breaches

If you do nothing else on this page, do these.

```text
 1. MFA on every administrative account, and on the cloud root account
 2. No long-lived cloud access keys — roles and OIDC instead
 3. No public storage buckets — block public access at the ACCOUNT level
 4. No database or admin interface reachable from the internet
 5. No secrets in code, config files, CI logs or environment dumps
 6. Audit logging on, and shipped to somewhere the attacker cannot delete
 7. Immutable backups, in a separate account, restore TESTED
 8. Dependencies scanned, and actually updated
 9. Parameterised queries and output encoding — no string-built SQL or HTML
10. Authorisation checked on every request, not just authentication
```

> [!IMPORTANT]
> **Real breaches are overwhelmingly caused by misconfiguration, not by sophisticated attacks.** A public bucket, an over-permissive role, a key in a repository, a database open to the world. This is genuinely good news: every one of those is preventable by a setting you can change this afternoon, and none of them require a security team. Fix the ten above before considering any product.

> [!CAUTION]
> **Buying a scanner that reports the same ten findings for two years has not improved anything.** Tools are for finding what you did not know about; they are not a substitute for fixing what you already know about. Work the list first, then instrument to keep it fixed.

---

# ─────────── COPY FROM HERE ───────────

## 1. Identity and access

- [ ] **MFA on every human account** with administrative access
- [ ] **Cloud root / global admin account**: hardware MFA, no access keys, effectively unused
- [ ] **No long-lived access keys** — instance roles for services, OIDC for CI, SSO for humans
- [ ] **Least privilege** — no `Action: *` / `Resource: *` outside a justified admin role
- [ ] **No shared accounts**; every action attributable to a person
- [ ] **Joiners and leavers process** — access removed on the day someone leaves
- [ ] **Access reviewed periodically** — stale write access accumulates silently
- [ ] **Separate accounts or projects** for production, staging and development
- [ ] **Guardrails above IAM** (SCPs or equivalent) that even an admin cannot bypass
- [ ] **Break-glass credentials** stored offline, MFA-protected, and alarmed on use

## 2. Network exposure

- [ ] **Nothing public that does not need to be** — databases, caches, brokers, admin UIs, dashboards
- [ ] **No `0.0.0.0/0`** on any port except 80/443 on a load balancer
- [ ] **No inbound SSH** — use a session manager that logs and needs no open port
- [ ] **Private subnets by default**; a public address requires a reason
- [ ] **Internal rules reference security groups**, not IP ranges
- [ ] **Egress restricted** where feasible — exfiltration goes outbound
- [ ] **TLS everywhere**, including internally; TLS 1.2 minimum
- [ ] **Certificates renewed automatically**, with a 30-day alert
- [ ] **Rate limiting and DDoS protection at the edge**

## 3. Secrets

- [ ] **No secrets in code, `.env` files in the repository, Dockerfiles, or CI logs**
- [ ] **A secret manager** with access control and an audit trail
- [ ] **Secret scanning in CI, with push protection**, so a leak is prevented rather than reported
- [ ] **Anything ever committed is rotated** — deleting the commit does nothing
- [ ] **Rotation is possible and has been done at least once**
- [ ] **Encryption keys recoverable independently** of the system they protect
- [ ] **No secrets in Kubernetes Secrets alone** — they are base64, not encryption
- [ ] **No secrets in Terraform state** treated as non-sensitive; state contains plaintext

## 4. Data protection

- [ ] **Encryption at rest** on every store — databases, disks, object storage, backups
- [ ] **Encryption in transit** everywhere
- [ ] **Personal data inventoried** — what you hold, where, and why
- [ ] **Retention defined and enforced** — data you no longer need is pure liability
- [ ] **Deletion actually deletes** — including from replicas, backups, caches and vector indexes
- [ ] **No production data in development or test environments** — or masked if unavoidable
- [ ] **Passwords hashed with argon2id, scrypt or bcrypt** — never a fast hash
- [ ] **Card data never touches your systems** — tokenise through a provider
- [ ] **Sensitive data not in logs, error messages, analytics or third-party tooling**

## 5. Application

- [ ] **All input validated server-side** — client validation is a convenience, not a control
- [ ] **Parameterised queries** — no SQL built by string concatenation
- [ ] **Output encoding** appropriate to context (HTML, attribute, JS, URL)
- [ ] **Content Security Policy** set, and not `unsafe-inline`
- [ ] **Authorisation checked on every request** — including object-level: can user A read record B?
- [ ] **CSRF protection** on state-changing requests where cookies are used
- [ ] **Session cookies:** `HttpOnly`, `Secure`, `SameSite`; no tokens in `localStorage`
- [ ] **No SSRF** — validate and allowlist any URL the server will fetch
- [ ] **File uploads:** type verified by content, size limited, stored outside the web root, never executed
- [ ] **Deserialisation** of untrusted input avoided
- [ ] **No secrets, versions or stack traces in error responses**
- [ ] **Security headers:** HSTS, `X-Content-Type-Options`, `Referrer-Policy`, frame options

## 6. Dependencies and supply chain

- [ ] **Dependency scanning** in CI, failing on critical findings
- [ ] **Updates actually applied** — an ignored alert queue is not a control
- [ ] **Lock files committed**, so builds are reproducible
- [ ] **Base images pinned by digest** and rebuilt regularly — an old image is an unpatched one
- [ ] **CI actions and plugins pinned to a commit SHA**, not a mutable tag
- [ ] **CI cannot be poisoned by a fork** — no secrets exposed to untrusted pull requests
- [ ] **Artefacts signed**, and signatures verified at deploy
- [ ] **New dependencies reviewed** — is it maintained, and does it need to exist?

## 7. Logging, detection and response

- [ ] **Audit logging enabled** on the cloud control plane and the application
- [ ] **Logs shipped to a separate account** an intruder cannot delete
- [ ] **Log retention set** — long enough to investigate, short enough to be lawful
- [ ] **Alerts on security events:** root usage, permission changes, disabled logging, authentication failure spikes, impossible travel
- [ ] **Someone actually receives the alerts**, and they are few enough to be read
- [ ] **An incident plan that includes revoking sessions**, not just disabling keys
- [ ] **Someone knows how to preserve evidence** before rebuilding
- [ ] **Contact route for external vulnerability reports** — a `SECURITY.md` and a monitored address

## 8. Backup and recovery

- [ ] **Backups exist** for everything that cannot be regenerated
- [ ] **Immutable** — Object Lock or equivalent, in a separate account
- [ ] **The credential that manages production cannot delete the backups**
- [ ] **A restore has been performed and timed** — this is the only proof they work
- [ ] **Retention long enough** for slow corruption to be noticed (30 days minimum for transactional data)
- [ ] **Encryption keys for backups recoverable** without the lost environment
- [ ] **Runbook stored outside** the environment it recovers

## 9. People and process

- [ ] **Nobody can deploy to production alone without review** — including administrators
- [ ] **Branch protection includes administrators**
- [ ] **Security review for changes touching auth, payments, permissions or personal data**
- [ ] **Offboarding removes access same-day**, including SaaS and repositories
- [ ] **Phishing is the main entry route** — MFA that resists it (passkeys, hardware keys) where possible
- [ ] **Support processes threat-modelled** — account recovery by a helpful human is a bypass
- [ ] **Third-party vendors reviewed** — their breach is your breach to your customers

# ─────────── COPY TO HERE ───────────

---

## Threat model in four questions

Before a detailed review, answer these about the system in front of you.

```text
1. WHAT would an attacker want?
   money · personal data · compute · access to your customers · disruption

2. WHO is the realistic attacker?
   opportunistic scanners (most common) · fraudsters · an insider
   · a targeted actor (rare, and a different budget)

3. WHERE is the boundary?
   every place untrusted input enters, and every place a credential is used

4. IF they succeed, HOW FAR do they get?
   ← this is the question worth the most engineering
```

> [!IMPORTANT]
> **Question 4 is where the real design work is, because prevention eventually fails somewhere.** Assume an application compromise happens: does it reach one tenant's data or all of them? Can it delete the audit log? Can it reach the backups? The difference between an incident and a catastrophe is blast radius, and blast radius is architecture — separate accounts, scoped credentials, immutable logs.

---

## What to do when something happens

```text
1. CONTAIN         revoke sessions and credentials; isolate, do not power off
                   → powering off destroys memory evidence
2. PRESERVE        snapshot before rebuilding; the logs are your only record
3. ASSESS          what did the credential have access to? CloudTrail will tell you
4. ERADICATE       rotate everything the attacker could have seen; rebuild from code
5. RECOVER         from immutable backups, into a properly configured environment
6. LEARN           blameless review; the finding is never "a person was careless"
7. NOTIFY          legal obligations have deadlines — know yours in advance
```

> [!CAUTION]
> **Rebuilding in a hurry is how a second incident is created.** A hastily reconstructed environment with open security groups and relaxed permissions is exactly what the attacker would want next. Rebuild from infrastructure code so the configuration is the reviewed one, not the improvised one.

---

## Common mistakes

- **Believing the cloud provider secures your data** — they secure their infrastructure
- **One account for everything**, so every mistake is a production mistake
- **Long-lived access keys**, which eventually reach a repository or a laptop backup
- **Authentication checked but not authorisation** — the most common application flaw
- **Secrets "removed" in a later commit** instead of rotated
- **Backups in the same account**, deletable by the same credentials
- **Audit logs in the account they audit**
- **A scanner bought instead of the findings fixed**
- **Compliance mistaken for security** — the overlap is partial
- **Alerts nobody reads**, producing false confidence
- **No egress restriction**, so exfiltration is trivial once inside
- **Branch protection that exempts administrators**
- **The support and account-recovery path never threat-modelled**
- **A restore never tested**, so the recovery time is unknown

---

## Personal Notes

*Your notes on using this checklist.*

- **Items that failed the first time I ran this:**
- **Our system's specific risks not on this list:**
- **What we automated so it stays fixed:**

---

## Workbook Exercise

- [ ] Walk the ten items at the top and mark each pass or fail, honestly. Fix the failures this week.
- [ ] Run an open-source cloud auditor (Prowler, ScoutSuite) and read only the high findings.
- [ ] Search your repository history for secrets. Rotate anything you find.
- [ ] Verify your audit logs cannot be deleted by a production credential.
- [ ] Attempt to read another user's record by changing an id. If it works, that is today's priority.
- [ ] Write down what you would do in the first ten minutes if a key leaked. If you cannot, that is the gap.
