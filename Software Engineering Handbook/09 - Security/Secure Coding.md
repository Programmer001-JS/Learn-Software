# Secure Coding

> **In one line —** the daily habits that prevent vulnerabilities, rather than the audit that finds them after they shipped.

| | |
|---|---|
| **Category** | Practice |
| **Architectural Layer** | Whole application |
| **Related notes** | [OWASP Top 10](OWASP%20Top%2010.md) · [Validation](../07%20-%20Backend%20Design%20Patterns/Validation.md) · [Authorization](Authorization.md) · [Secrets Management](Secrets%20Management.md) · [Code Review Checklist](../16%20-%20Templates/Code%20Review%20Checklist.md) |

---

## 1. Short Definition

*What is it?*

Secure coding is the set of defaults and habits applied while writing code so that common vulnerability classes cannot occur — as opposed to security testing, which discovers them afterwards.

---

## 2. Purpose

*What is its main purpose?*

To move security left. A vulnerability prevented at write time costs minutes; the same vulnerability found in production costs an incident, a disclosure and a patch cycle.

---

## 3. The principles

```text
NEVER TRUST INPUT           validate at every boundary
FAIL CLOSED                 deny by default; errors must not grant access
LEAST PRIVILEGE             minimum permissions, everywhere
DEFENCE IN DEPTH            assume any single control will fail
KEEP IT SIMPLE              complexity hides bugs
DON'T ROLL YOUR OWN CRYPTO  use vetted libraries, always
```

---

## 4. Input — validate, then escape

Two different jobs, frequently confused:

```text
VALIDATION   at INPUT   — is this acceptable?      → reject
ESCAPING     at OUTPUT  — is this safe HERE?       → transform
```

```text
Same value, three destinations, three different escapes:
    HTML page      → HTML-escape
    SQL query      → parameterise (not escape)
    Shell command  → argument array, never string interpolation
    URL            → URL-encode
```

> [!IMPORTANT]
> **Escaping is context-specific.** "Sanitising once at input" is a comforting idea that does not work — the same string is dangerous in different ways depending on where it ends up.

---

## 5. Injection — one shape, many forms

```python
# SQL
cursor.execute(f"... WHERE id = {user_id}")            # ✗
cursor.execute("... WHERE id = %s", (user_id,))        # ✓

# Shell
os.system(f"convert {filename} out.png")               # ✗
subprocess.run(["convert", filename, "out.png"])       # ✓  no shell at all

# Path
open(f"/uploads/{filename}")                           # ✗  ../../etc/passwd
open(safe_join("/uploads", filename))                  # ✓
```

> [!TIP]
> The rule underneath all three: **never build a command, query or path by concatenating untrusted input.** Pass data as data — parameters, argument arrays, resolved paths.

---

## 6. Output — XSS

```text
✗ innerHTML = userInput
✗ {!! $userInput !!}  ·  {{ user_input|safe }}
✓ textContent = userInput
✓ Template auto-escaping, left ON
✓ Content-Security-Policy as the structural backstop
```

> [!CAUTION]
> Every template engine escapes by default and offers a way to disable it. Every `|safe`, `|raw` and `dangerouslySetInnerHTML` in a codebase should have a written reason next to it.

---

## 7. Errors and logging

```text
✗ return str(exception)                 leaks paths, versions, SQL
✗ log.info(f"login: {request.body}")    captures passwords in log files
✓ return { "error": "invalid_request", "request_id": "req_8f3a" }
✓ log the detail server-side, keyed by request_id
```

```text
NEVER LOG:  passwords · tokens · API keys · card numbers
            full request bodies · full cookie headers
```

---

## 8. Secrets

```text
✗ API_KEY = "sk_live_abc123"       in source
✗ .env committed to git
✓ environment variables from a secrets manager
✓ .env in .gitignore, with .env.example checked in
✓ secret scanning in CI
```

> [!CAUTION]
> **A secret committed to git is compromised permanently**, even after the commit is removed — the history is cloned, cached and often mirrored. Rotate rather than delete. See [Secrets Management](Secrets%20Management.md).

---

## 9. Dependencies

```text
Before adding a package:
    Is it maintained? How many transitive dependencies does it pull in?
    Is the functionality worth the supply-chain risk?
    ↓
After adding it:
    lockfile committed · automated scanning · a patching policy
```

> [!IMPORTANT]
> A typical Node project has hundreds of transitive dependencies, **every one of which runs with your process's full privileges**. This is now one of the largest realistic attack surfaces in application development.

---

## 10. Safe defaults

| Setting | Secure default |
|---|---|
| **New endpoint** | Requires authentication and authorisation |
| **New database column** | Not returned by the API unless explicitly added |
| **New file upload** | Type and size restricted |
| **CORS** | Specific origins, never `*` on an authenticated API |
| **Cookies** | `HttpOnly`, `Secure`, `SameSite` |
| **Errors** | Generic to the client, detailed in logs |

> [!TIP]
> **Design so the secure path is the easy path.** A base controller that requires an explicit authorisation call, a serialiser that opts fields in rather than out — these prevent whole classes of mistake without relying on anyone remembering.

---

## 11. Real World Example

- **Parameterised queries** eliminate SQL injection entirely, which is why frameworks made them the default rather than a recommendation.
- **Content Security Policy** limits the damage of an XSS that slipped through — defence in depth working as intended.
- **Secret scanning** in CI catches committed keys before they reach a public repository, which is a routine occurrence.

---

## 12. When To Use / When NOT To Use

> [!TIP]
> Apply these defaults everywhere, always. They cost nothing at write time and are expensive to retrofit.

> [!CAUTION]
> Security is not free of trade-offs, and pretending otherwise leads to it being switched off. Strict CSP breaks inline scripts, aggressive rate limiting frustrates legitimate users, and short sessions annoy people. Make those trade-offs consciously and write down the reasoning — an undocumented security control is one that will be removed by someone who does not know why it exists.

---

## 13. Advantages and Disadvantages

**Advantages**
- Prevents whole vulnerability classes rather than individual bugs
- Costs almost nothing during development
- Makes code review faster — deviations stand out
- Reduces incident and audit burden considerably

**Disadvantages**
- Some friction in daily work
- Requires team-wide consistency; one gap is enough
- Over-applied, it produces security theatre that nobody follows
- Cannot prevent business-logic flaws, which need thinking rather than rules

---

## 14. Security Considerations

Beyond the categories above, three that catch experienced developers:

- **Race conditions** — check-then-act on balances, coupons or stock is exploitable. Use [transactions](../08%20-%20Databases%20and%20Data/Transactions.md) and database constraints
- **Timing side channels** — comparing secrets with early-exit equality leaks information. Use constant-time comparison for tokens and hashes
- **SSRF** — any feature fetching a user-supplied URL must validate the *resolved* address, not the string

---

## 15. Mental Model

> [!NOTE]
> **Secure coding is kitchen hygiene, not a food inspection.**
>
> Inspections find problems after the fact. Hygiene — separate boards, washed hands, checked temperatures — means most problems never occur. Neither replaces the other, and no amount of inspection saves a kitchen that does not wash anything.

---

## 16. Mini Architecture Diagram

```text
Untrusted input
    ↓
VALIDATE     allow-list schema, size limits
    ↓
AUTHENTICATE · AUTHORISE (object-level)
    ↓
Business logic — parameterised queries, no shell strings, no path building
    ↓
ESCAPE for the destination context
    ↓
Response: generic errors, no stack traces, security headers
    ↓
Log the event without logging the secrets
```

---

## 17. Complete Request Flow

```text
Request arrives
    ↓
Body size limited before parsing        ← resource exhaustion defence
    ↓
Schema validation, allow-list           → 422 with field errors
    ↓
Authentication                          → 401
    ↓
Object-level authorisation              → 403 or 404
    ↓
Parameterised database access
    ↓
Response built from an explicit DTO — never the ORM entity
    ↓
Security headers applied; errors generic; request_id logged
    ↓
An exception anywhere → generic 500 to the client, full detail in logs
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> Validate at input, escape at output, parameterise everything, fail closed, and never build a query, command or path from untrusted strings.

---

## 19. Common Mistakes

- **Sanitising once at input** instead of escaping per output context
- **String-built SQL, shell commands or file paths**
- **Disabling template escaping** for convenience
- **Returning exception details** to clients
- **Logging request bodies** containing credentials
- **Committing secrets**, then deleting the commit instead of rotating the key
- **Adding dependencies casually**
- **Checking authentication but not object ownership**

---

## 20. Open Source Technologies

- **Semgrep**, **Bandit**, **CodeQL**, **ESLint security plugins** — static analysis
- **OWASP ZAP** — dynamic scanning
- **gitleaks**, **trufflehog** — secret scanning in CI
- **Trivy**, **Dependabot**, **Snyk** — dependency scanning
- **DOMPurify**, **bleach** — HTML sanitisation when it is genuinely required
- **OWASP Cheat Sheet Series** — the practical reference for each topic here

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Search your codebase for string-formatted SQL, `os.system`, and disabled template escaping.
- [ ] Check whether any log statement could capture a password or token.
- [ ] Add one static analysis tool to CI and triage what it reports.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Input → validate → authenticate → authorise → parameterised access → escape → log
```

## 2. Request Flow

```text
Input       untrusted data from anywhere
    ↓
Processing  validated, authorised, handled without concatenation
    ↓
Output      escaped for its destination, with errors that reveal nothing
```

## 3. Real-World Usage

**Parameterised queries** are the clearest success story in application security: once frameworks made them the default rather than the advice, SQL injection went from ubiquitous to a notable finding. Good defaults beat good intentions.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Habits and defaults that prevent vulnerability classes at write time |
| **Why does it exist?** | Because prevention is orders of magnitude cheaper than remediation |
| **Where does it belong?** | In every file, every review, every default |
| **When should I use it?** | Always — with trade-offs made consciously and written down |
