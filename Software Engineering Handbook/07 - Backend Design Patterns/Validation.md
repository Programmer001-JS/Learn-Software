# Validation

> **In one line —** rejecting bad input at the boundary, because every request is hostile until proven otherwise — including the ones from your own frontend.

| | |
|---|---|
| **Category** | Architectural Pattern |
| **Architectural Layer** | Application boundary |
| **Related notes** | [Middleware](Middleware.md) · [Controllers](Controllers.md) · [API Design](../04%20-%20Networking%20and%20Internet/API%20Design.md) · [Secure Coding](../09%20-%20Security/Secure%20Coding.md) · [TypeScript](../05%20-%20Frontend%20Architecture/TypeScript.md) |

---

## 1. Short Definition

*What is it?*

Validation is checking that incoming data has the expected shape, types and ranges — and rejecting it with a clear error if it does not — before any business logic runs.

---

## 2. Purpose

*What is its main purpose?*

To guarantee that everything past the boundary is well-formed, so the rest of your code can stop defending itself and just work.

---

## 3. Problem

*What engineering problem does it solve?*

```text
WITHOUT VALIDATION AT THE EDGE

Request: {"age": "abc", "email": null}
    ↓
Controller: no check
    ↓
Service: no check
    ↓
Repository: no check
    ↓
Database: constraint violation, or worse — it accepts it
    ↓
A 500 error, or corrupt data discovered three weeks later
```

Every layer either re-checks (duplication) or assumes someone else did (disaster). Validating once, at the boundary, resolves both.

---

## 4. Architecture Position

```text
Request
    ↓
Middleware
    ↓
Routing
    ↓
━━━ VALIDATION ━━━     ← the boundary; everything past this point is trusted
    ↓
Controller
    ↓
Service   (business RULES — a different kind of checking)
    ↓
Repository → database  (constraints as the last line of defence)
```

---

## 5. The three kinds of checking, which are not the same

> [!IMPORTANT]
> These are constantly conflated, and separating them clarifies where each belongs.

| | Question | Where | Failure |
|---|---|---|---|
| **Validation** | Is this well-formed? | The boundary | `400` / `422` |
| **Business rules** | Is this allowed *given our domain*? | The service layer | `409` / `422` |
| **Authorisation** | May *this user* do this? | Middleware or service | `403` |

```text
"email must be a valid address"        → validation
"you cannot cancel a shipped order"    → business rule
"this order is not yours"              → authorisation
```

---

## 6. Schema-driven validation

Declare the shape once; the library does the checking and gives you a typed object.

```python
class CreateOrder(BaseModel):
    email: EmailStr
    quantity: int = Field(gt=0, le=100)
    notes: str | None = Field(default=None, max_length=500)
```

```typescript
const CreateOrder = z.object({
  email: z.string().email(),
  quantity: z.number().int().positive().max(100),
  notes: z.string().max(500).optional(),
});
```

> [!TIP]
> Schema-based validation is better than hand-written `if` statements because the schema is also documentation, also a type definition, and cannot drift out of sync with the checks.

---

## 7. Allow-list, never deny-list

```text
DENY-LIST  "reject anything containing <script>"
           ↓ you will always miss a case; attackers only need one

ALLOW-LIST "accept only a-z, 0-9, and hyphens, max 50 characters"
           ↓ everything unexpected is rejected by default
```

> [!CAUTION]
> Every input filter built by blocking known-bad patterns has eventually been bypassed. Define what is acceptable and reject everything else.

---

## 8. Client-side validation is not validation

> [!CAUTION]
> Browser validation is a **user-experience feature**. Anyone can send a request with `curl`, and mobile clients, scripts and other services never touched your form at all. **Every rule must be enforced on the server**, without exception.

```text
Client validation   → fast feedback, better UX     → optional
Server validation   → correctness and security     → mandatory
Database constraints → last line of defence        → strongly recommended
```

---

## 9. Real World Example

- **[FastAPI](../06%20-%20Backend%20Architecture/FastAPI.md)** derives validation, docs and types from one Pydantic model.
- **Zod** in TypeScript both validates at runtime and infers the static type — closing the gap [TypeScript](../05%20-%20Frontend%20Architecture/TypeScript.md) leaves open.
- **Django Forms and DRF serialisers** validate and coerce in one step.
- **SQL injection** is prevented by parameterised queries, but validation is what stops nonsense reaching the query in the first place.

---

## 10. Good error responses

```json
{
  "error": "validation_failed",
  "message": "Request contains invalid fields",
  "fields": [
    { "field": "email", "message": "must be a valid email address" },
    { "field": "quantity", "message": "must be between 1 and 100" }
  ],
  "request_id": "req_8f3a"
}
```

> [!TIP]
> Return **all** validation errors at once, not the first one. A client that must submit five times to discover five problems is a bad API. And use `422` for a well-formed request with invalid content, `400` for malformed syntax.

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Validate at **every trust boundary**: HTTP requests, queue messages, webhook payloads, file uploads, environment variables, and responses from third-party APIs.

> [!CAUTION]
> Do not validate the same thing at every internal layer — it duplicates rules that then drift apart. Validate once at the edge, then trust your own types. And do not put business rules in the validation schema; "quantity must be positive" is validation, "not enough stock" is a domain rule needing a database check.

---

## 12. Advantages and Disadvantages

**Advantages**
- Bad data never reaches business logic
- Clear, actionable errors for clients
- Schemas double as documentation and types
- Removes an entire class of runtime failures

**Disadvantages**
- Some upfront work per endpoint
- Schemas can drift from reality if not derived from one source
- Over-validating internal calls adds friction without benefit
- Poor error messages are worse than none

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **CPU** | Small; Pydantic v2 and Zod are fast |
| **Early rejection** | **Saves** work — a bad request never reaches the database |
| **Body size limits** | Enforce before parsing, or you have already paid the cost |

---

## 14. Security Considerations

> [!CAUTION]
> **Mass assignment** is the validation failure with the widest impact: binding a request body directly to a model lets a caller set fields you never intended to expose — `is_admin`, `role`, `account_balance`. Always bind to an explicit input schema listing exactly the accepted fields.

- **Validation is not sanitisation** — validation decides *whether to accept*; escaping on output prevents XSS
- **Validation is not authorisation** — a perfectly valid request can still be forbidden
- **Set size limits** on strings, arrays and request bodies, or you have a memory-exhaustion vector
- **Beware catastrophic regexes** — a badly written pattern on hostile input hangs the process (ReDoS)
- **Validate file uploads** by content type and size, never by filename extension
- **Validate URLs** before fetching them — otherwise you have an SSRF vulnerability

---

## 15. Mental Model

> [!NOTE]
> **Validation is passport control at the border.**
>
> Everyone is checked once, at the edge, against a clear list of requirements. Once inside, nobody re-checks passports at every shop — because the border did its job. And the guard checks that the document is *valid*, not whether you are *welcome*; that is a different question, asked by someone else.

---

## 16. Mini Architecture Diagram

```text
Untrusted input
    ↓
┌────── VALIDATION ──────┐
│  shape                 │
│  types                 │
│  ranges and lengths    │
│  allowed values        │
└───────────┬────────────┘
   fail → 422 with field errors
            ↓ pass
      TRUSTED, typed object
            ↓
      Controller → Service (business rules) → Repository
            ↓
      Database constraints (last line of defence)
```

---

## 17. Complete Request Flow

```text
POST /orders  {"email": "x", "quantity": -5}
    ↓
Body size limit checked BEFORE parsing
    ↓
Parsed as JSON
    ↓
Schema validation:
    email    → invalid format
    quantity → must be positive
    ↓
422 with both errors and a request_id — no database call made
    ↓
─────────── on a valid request instead ───────────
    ↓
Typed CreateOrder object passed to the controller
    ↓
Service checks BUSINESS rules: is there stock? → 409 if not
    ↓
Repository writes; database constraints catch anything that slipped through
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> Validate once at the boundary with an explicit allow-list schema — client-side checks are UX, and every rule must be enforced again on the server.

---

## 19. Common Mistakes

- **Trusting client-side validation**
- **Mass assignment** — binding a request body directly to a database model
- **Deny-lists** instead of allow-lists
- **Returning only the first error**
- **Confusing validation with authorisation**
- **No length or size limits**, enabling memory exhaustion
- **Validating in the controller by hand** instead of declaratively
- **Not validating queue messages, webhooks or third-party responses** — they are boundaries too

---

## 20. Open Source Technologies

- **Pydantic** (Python), **Zod**, **Valibot**, **Joi** (TypeScript)
- **class-validator** (NestJS), **FluentValidation** (.NET)
- **Bean Validation / Hibernate Validator** (Java)
- **Django Forms**, **DRF serialisers**
- **JSON Schema**, **OpenAPI** — language-neutral schema definitions

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Find one endpoint that binds a request body directly to a database model and introduce an input schema.
- [ ] Send a deliberately invalid request to your API and judge whether the error response is actually usable.
- [ ] Check that queue messages and webhook payloads in your system are validated as strictly as HTTP requests.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Untrusted input
    ↓
Validation (boundary)
    ↓
Controller → Service (business rules) → Repository
    ↓
Database constraints
```

## 2. Request Flow

```text
Input       raw, untrusted data
    ↓
Processing  checked against an explicit schema; rejected with field-level errors
    ↓
Output      a typed, trusted object — or a 422 before any work is done
```

## 3. Real-World Usage

**FastAPI and Zod** both made validation and typing the same declaration. That convergence — one schema serving as validator, type and documentation — is the current best practice in both ecosystems.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Checking incoming data against an explicit schema at the boundary |
| **Why does it exist?** | Because all external input is untrusted and bad data spreads |
| **Where does it belong?** | At every trust boundary, before business logic |
| **When should I use it?** | Always on the server — client-side checks are convenience only |
