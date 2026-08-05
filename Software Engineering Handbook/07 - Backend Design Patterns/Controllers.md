# Controllers

> **In one line —** the thin translation layer between HTTP and your application; when it grows fat, everything below it becomes untestable.

| | |
|---|---|
| **Category** | Architectural Pattern |
| **Architectural Layer** | Application, HTTP boundary |
| **Also called** | Handler · Resource · View (Django) · Endpoint |
| **Related notes** | [Services](Services.md) · [Routing](Routing.md) · [Validation](Validation.md) · [Repository Pattern](Repository%20Pattern.md) · [Clean Architecture](Clean%20Architecture.md) |

---

## 1. Short Definition

*What is it?*

A controller receives a request, converts it into a call to your application, and converts the result back into an HTTP response. That is its entire job.

---

## 2. Purpose

*What is its main purpose?*

To keep HTTP knowledge in one place, so that business logic knows nothing about status codes, headers or request objects — and can therefore be tested, reused and called from anywhere.

---

## 3. Problem

*What engineering problem does it solve?*

```text
FAT CONTROLLER                        THIN CONTROLLER

def create_order(request):            def create_order(body: CreateOrder, user):
    data = request.json                   order = order_service.create(user, body)
    if not data.get('items'): ...         return 201, OrderDto.from(order)
    user = db.query(User)...
    stock = db.query(Stock)...        ← HTTP in, HTTP out. Nothing else.
    if stock < qty: ...
    total = sum(...)                  Business logic lives in the service,
    if user.tier == 'gold': ...       where it can be tested without HTTP,
    order = Order(...)                reused by a CLI or a queue consumer,
    db.add(order); db.commit()        and read without wading through
    send_email(...)                   request parsing.
    return jsonify(...)
```

> [!IMPORTANT]
> Every framework tutorial writes the fat version, because it fits on one slide. Every codebase that kept writing it eventually cannot test anything without spinning up an HTTP server.

---

## 4. Architecture Position

```text
Request
    ↓
Middleware  →  Routing  →  Validation
    ↓
CONTROLLER      ← translate HTTP → application call
    ↓
Service         ← business logic, no HTTP knowledge
    ↓
Repository      ← data access
    ↓
Database
```

---

## 5. What belongs in a controller

**Yes:**
- Reading path, query and body parameters
- Calling **one** service method
- Mapping the result to a response DTO
- Choosing the status code (`200`, `201`, `204`, `404`)
- Setting HTTP-specific headers (`Location`, `Cache-Control`)

**No:**
- Business rules and calculations
- Database queries
- Sending emails or calling third-party APIs
- Transaction management
- Anything you would want to reuse from a background job

> [!TIP]
> A useful test: **could this endpoint's behaviour be triggered from a CLI command or a queue consumer without rewriting it?** If not, logic is stuck in the controller.

---

## 6. The shape to aim for

```python
@router.post("/orders", status_code=201, response_model=OrderDto)
async def create_order(
    body: CreateOrder,                          # validated
    user: User = Depends(current_user),         # authenticated
    service: OrderService = Depends(),          # injected
):
    order = await service.create(user, body)    # ← the only real line
    return OrderDto.from_domain(order)
```

Five lines, no branching, no queries. Everything interesting happens one layer down.

---

## 7. Mapping errors to status codes

Services should raise **domain exceptions**, not HTTP errors. The controller layer — usually a global exception handler — translates them.

```text
Service raises                    Controller layer returns
────────────────────────────────────────────────────────
NotFoundError                     404
PermissionDeniedError             403
ValidationError                   422
ConflictError  (out of stock)     409
RateLimitError                    429
anything else                     500 + logged with a request ID
```

> [!IMPORTANT]
> A service that raises `HTTPException` is coupled to HTTP and cannot be reused by a scheduled job or a message consumer. Keep status codes on the HTTP side of the boundary.

---

## 8. Real World Example

- **Spring** `@RestController`, **NestJS** `@Controller`, **ASP.NET** `ControllerBase`, **Django** views, **FastAPI** route functions — the same role under different names.
- **Rails' "fat model, skinny controller"** was the original articulation of this idea.
- **A CLI command and an HTTP endpoint sharing one service** is the practical payoff, and the clearest sign the split is real.

---

## 9. Communication and Dependencies

- **[Routing](Routing.md)** dispatches to it
- **[Validation](Validation.md)** has already run, so it receives typed input
- **[Services](Services.md)** are what it calls
- **DTOs** are what it returns — never domain entities

> [!CAUTION]
> **Never return ORM entities directly.** They serialise every field, including ones added later by someone thinking about a different feature, and they can trigger lazy-loading during serialisation. Map to an explicit response DTO — this is a data-leak defence, not just tidiness.

---

## 10. When To Use / When NOT To Use

> [!TIP]
> Keep controllers thin from the very first endpoint. Extracting logic later is far more work than putting it in the right place initially.

> [!CAUTION]
> The counter-argument is real: for a genuinely trivial CRUD service, a service layer that only forwards calls to a repository is pure ceremony. Judge by whether there are business rules to protect — if `create_order` really is one insert with no rules, a thin controller calling the repository directly is honest.

---

## 11. Advantages and Disadvantages

**Advantages**
- Business logic testable without HTTP
- Logic reusable from CLI, jobs and queue consumers
- Controllers stay readable — the endpoint's intent is visible at a glance
- Swapping REST for gRPC touches only this layer

**Disadvantages**
- More files and more indirection
- Over-applied to trivial CRUD, it becomes pass-through noise
- DTO mapping is genuine boilerplate

---

## 12. Performance Impact

Negligible — this is a structural concern, not a performance one. The one real cost is **DTO mapping**, which is trivial except in very high-throughput serialisation paths.

---

## 13. Security Considerations

> [!CAUTION]
> **The controller is where object-level authorisation is most often forgotten.** Middleware confirmed the user is authenticated; nothing has confirmed that order 42 belongs to *them*. This is the single most common serious API vulnerability.

```python
order = await service.get(order_id)
if order.user_id != user.id:          # ← this check, or its absence, is the bug
    raise PermissionDeniedError()
```

- **Return DTOs, never entities** — the most common accidental data-leak path
- **Do not leak internals in errors** — map exceptions centrally, never return stack traces
- **Validate before the controller**, so it receives only well-formed input
- **Return 404 rather than 403** where the existence of a resource is itself sensitive

---

## 14. Mental Model

> [!NOTE]
> **A controller is a receptionist.**
>
> They take your request, translate it into whatever the office needs, pass it to the right specialist, and hand you back the answer in a form you understand. A receptionist who starts doing the specialist's work themselves is exactly the fat controller problem — and now nobody else can do that work either.

---

## 15. Mini Architecture Diagram

```text
HTTP request
    ↓
Middleware → Routing → Validation
    ↓
┌──────── CONTROLLER ────────┐
│  extract parameters        │
│  call ONE service method   │
│  map result → DTO          │
│  choose the status code    │
└─────────────┬──────────────┘
              ↓
      Service (no HTTP knowledge)
              ↓
      Repository → database
              ↓
   Exception handler maps domain errors → status codes
```

---

## 16. Complete Request Flow

```text
POST /orders
    ↓
Middleware: auth → user identified
    ↓
Routing → controller matched
    ↓
Validation → typed CreateOrder object
    ↓
CONTROLLER:
    order = await order_service.create(user, body)
    ↓
    SERVICE: stock check, pricing, discount rules, transaction,
             publish OrderCreated event
    ↓
    REPOSITORY: insert
    ↓
Service returns a domain Order
    ↓
CONTROLLER maps it to OrderDto — only declared fields leave
    ↓
201 Created, Location: /orders/42
    ↓
Service raised ConflictError instead? → global handler → 409
```

---

## 17. Key Takeaway

> [!IMPORTANT]
> A controller translates HTTP to an application call and back — if it contains business logic, that logic cannot be tested, reused or moved.

---

## 18. Common Mistakes

- **Business logic in the controller** — the defining failure
- **Database queries in the controller**
- **Returning ORM entities** instead of DTOs
- **Raising HTTP exceptions from services**, coupling the domain to HTTP
- **Forgetting object-level authorisation**
- **Controllers calling several services and orchestrating them** — that orchestration is itself business logic
- **Catching exceptions per-endpoint** instead of handling them centrally

---

## 19. Open Source Technologies

- **FastAPI**, **Django REST Framework**, **Flask** — Python
- **NestJS**, **Express** — TypeScript
- **Spring `@RestController`**, **ASP.NET `ControllerBase`**
- **MapStruct**, **AutoMapper**, **Pydantic** — DTO mapping
- **Exception handling**: `@ControllerAdvice`, exception filters, `@app.exception_handler`

---

## 20. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 21. Workbook Exercise

- [ ] Take your longest controller method and list which lines are HTTP concerns and which are business logic.
- [ ] Check whether any endpoint returns an ORM entity directly.
- [ ] Pick one endpoint and ask whether a scheduled job could reuse its logic without rewriting it.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Request → Middleware → Routing → Validation → Controller → Service → Repository → DB
```

## 2. Request Flow

```text
Input       a validated, typed request plus an authenticated user
    ↓
Processing  one service call, then mapping the result to a DTO
    ↓
Output      a status code and a response body containing only declared fields
```

## 3. Real-World Usage

**Rails' "skinny controller, fat model"** became a maxim because early Rails applications put everything in controllers and became untestable. Every framework since has repeated the lesson under a different vocabulary.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | The translation layer between HTTP and your application |
| **Why does it exist?** | So business logic is free of HTTP and therefore testable and reusable |
| **Where does it belong?** | Between routing and the service layer |
| **When should I use it?** | Every HTTP application — kept thin from the first endpoint |
