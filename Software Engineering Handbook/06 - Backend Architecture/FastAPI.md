# FastAPI

> **In one line —** a Python framework where type hints do triple duty: validation, serialisation and automatic API documentation.

| | |
|---|---|
| **Category** | Web Framework |
| **Architectural Layer** | Application |
| **Language** | Python (ASGI) |
| **Built on** | Starlette + Pydantic |
| **Related notes** | [Backend Frameworks](Backend%20Frameworks.md) · [Uvicorn](Uvicorn.md) · [Validation](../07%20-%20Backend%20Design%20Patterns/Validation.md) · [Dependency Injection](../07%20-%20Backend%20Design%20Patterns/Dependency%20Injection.md) · [API Design](../04%20-%20Networking%20and%20Internet/API%20Design.md) |

---

## 1. Short Definition

*What is it?*

FastAPI is an async Python web framework that reads your **type hints** and uses them to validate incoming data, serialise outgoing data, and generate OpenAPI documentation — all from the same declaration.

---

## 2. Purpose

*What is its main purpose?*

To remove the gap between what an API claims to accept and what it actually accepts. In most frameworks the signature, the validation and the documentation are three separate things that drift apart. In FastAPI they are one thing.

---

## 3. Problem

*What engineering problem does it solve?*

```text
TRADITIONAL                            FASTAPI

def create_user(request):              @app.post("/users")
    data = request.json                async def create_user(user: UserIn) -> UserOut:
    if 'email' not in data: ...            return await service.create(user)
    if not valid_email(...): ...
    if 'age' not in data: ...          ← validation, parsing, docs, and types
    ...                                   all derived from one declaration
    # docs written separately
    # and now out of date
```

---

## 4. Architecture Position

```text
Nginx
    ↓
Gunicorn (process manager)
    ↓
Uvicorn worker (event loop)
    ↓
FASTAPI            ← routing, validation, dependency injection, serialisation
    ↓
Your services
    ↓
async database driver
```

---

## 5. The core mechanism

```python
class UserIn(BaseModel):
    email: EmailStr
    age: int = Field(ge=0, le=120)

@app.post("/users", response_model=UserOut)
async def create_user(user: UserIn, db: Session = Depends(get_db)):
    return await service.create(db, user)
```

From those few lines FastAPI derives:

- **Validation** — a malformed body returns `422` with per-field errors, automatically
- **Parsing** — the handler receives a typed object, not a dict
- **Serialisation** — `response_model` controls exactly what leaves, filtering internal fields
- **Documentation** — an OpenAPI schema and a browsable Swagger UI at `/docs`
- **Editor support** — real autocomplete on `user.email`

> [!IMPORTANT]
> `response_model` is a **security feature**, not just documentation. It defines exactly which fields are serialised, which is what stops a password hash or internal flag leaking because someone added a column to the ORM model.

---

## 6. Dependency injection

```python
async def get_current_user(token: str = Depends(oauth2_scheme)) -> User:
    ...

@app.get("/me")
async def me(user: User = Depends(get_current_user)):
    return user
```

Dependencies are ordinary functions, are cached per request, are overridable in tests, and can be nested. It is one of the cleanest DI systems in any dynamic language — see [Dependency Injection](../07%20-%20Backend%20Design%20Patterns/Dependency%20Injection.md).

---

## 7. Real World Example

- **AI and ML services** overwhelmingly use FastAPI — the model is Python, and the API layer needs to be too.
- **Microsoft, Uber and Netflix** have publicly described using it for internal services.
- **Any Python service exposed to a TypeScript frontend** benefits: the OpenAPI schema generates a typed client automatically.

---

## 8. Async — the caveat that matters most

> [!CAUTION]
> FastAPI is only fast if you do not block the [event loop](../05%20-%20Frontend%20Architecture/Event%20Loop.md). One synchronous call inside an `async def` handler stalls every concurrent request in that worker.

```python
async def bad():
    return requests.get(url)          # ✗ blocks the loop for everyone

async def good():
    return await client.get(url)      # ✓ httpx

def also_fine(): ...                  # ✓ plain def → FastAPI runs it in a thread pool
```

> [!TIP]
> If a library has no async version, declare the handler as plain `def`. FastAPI dispatches synchronous handlers to a thread pool automatically, which is safe — mixing blocking calls into `async def` is what breaks things.

---

## 9. Communication and Dependencies

- **[Uvicorn](Uvicorn.md)** — the ASGI server it runs on
- **Pydantic** — validation and serialisation
- **Starlette** — routing, middleware, WebSockets underneath
- **SQLAlchemy async / asyncpg** — database access
- **OpenAPI** — the generated contract

---

## 10. Alternatives

```text
FastAPI      async, type-driven, automatic docs, API-focused
    ↓
Django       batteries-included: ORM, admin, auth — far more than an API layer
    ↓
Flask        minimal, synchronous, huge ecosystem, no built-in validation
    ↓
Litestar     similar philosophy, more built-in structure
    ↓
Express/NestJS   the JavaScript equivalents
```

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Use FastAPI for JSON APIs, especially I/O-bound ones, and above all for anything serving ML models — Python is where the models are, and FastAPI is the best Python API layer.

> [!CAUTION]
> - **A full server-rendered web application** with admin, auth and templates is a better fit for [Django](Django.md); FastAPI gives you none of that.
> - **CPU-heavy request handling** will block the loop; move it to a [background worker](../10%20-%20Distributed%20Systems/Background%20Workers.md).
> - **A team unfamiliar with async Python** will produce blocking handlers and get none of the benefit.

---

## 12. Advantages and Disadvantages

**Advantages**
- Validation, serialisation and docs from one declaration
- Automatic, always-accurate OpenAPI and Swagger UI
- Excellent async performance for I/O-bound work
- Clean, testable dependency injection
- Real editor support through type hints

**Disadvantages**
- No ORM, admin, auth or migrations — you assemble them
- Async requires async libraries end to end
- Easy to accidentally block the event loop
- Younger ecosystem than Django's
- Pydantic v1→v2 was a genuinely disruptive migration

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Throughput** | Among the fastest Python frameworks, when non-blocking |
| **Concurrency** | Thousands of concurrent awaits per worker |
| **CPU** | Pydantic v2 validation is Rust-backed and fast |
| **Memory** | Modest, plus per-worker interpreter cost |

---

## 14. Security Considerations

> [!CAUTION]
> **Always set `response_model`.** Returning an ORM object directly serialises every attribute it has — including ones added later by someone who did not think about this endpoint. This is a common and quiet data-leak path.

- **Validation is not authorisation** — Pydantic checks shape, never permission
- **`Depends` for auth is only as good as the dependency**; verify object ownership inside handlers
- **CORS is off by default** — configure it deliberately rather than with `allow_origins=["*"]`
- **Disable `/docs` in production** if the API is not public
- **Never return exception details** to clients; map them to clean status codes

---

## 15. Mental Model

> [!NOTE]
> **FastAPI is a form with the rules printed on it.**
>
> The form itself states which fields are required and what shape they must take. Nobody has to write a separate rulebook, and the rulebook cannot fall out of date — because the form *is* the rulebook, and the same paper is handed to the person filling it in as to the person checking it.

---

## 16. Mini Architecture Diagram

```text
Client
    ↓
Nginx → Gunicorn → Uvicorn worker
    ↓
┌──────────── FastAPI ────────────┐
│  routing                        │
│  Pydantic validation → 422      │
│  Depends: auth, db session      │
│  handler                        │
│  response_model serialisation   │
└───────────────┬─────────────────┘
                ↓
        Service layer → repository
                ↓
        PostgreSQL / Redis
```

---

## 17. Complete Request Flow

```text
POST /users with a JSON body
    ↓
Uvicorn hands it to FastAPI
    ↓
Route matched
    ↓
Dependencies resolved: DB session, current user
    ↓
Pydantic validates the body → 422 with field errors if invalid
    ↓
Handler receives a typed UserIn object
    ↓
await service.create()  → coroutine suspends, worker serves others
    ↓
Result filtered through response_model — only declared fields leave
    ↓
201 Created; OpenAPI schema already documents all of this
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> FastAPI derives validation, serialisation and documentation from your type hints — and its speed depends entirely on never blocking the event loop.

---

## 19. Common Mistakes

- **Blocking calls in `async def` handlers**
- **Omitting `response_model`** and leaking internal fields
- **Treating validation as authorisation**
- **`allow_origins=["*"]`** on an authenticated API
- **Leaving `/docs` public** on an internal service
- **Mixing sync and async database sessions**
- **Putting business logic in the route handler**

---

## 20. Open Source Technologies

- **FastAPI**, **Starlette**, **Pydantic**
- **Uvicorn**, **Gunicorn** — serving
- **SQLAlchemy 2.0 async**, **asyncpg**, **SQLModel** — data access
- **Alembic** — migrations
- **openapi-typescript**, **openapi-generator** — generate typed clients from the schema

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Check every endpoint in a project for a `response_model` and add the missing ones.
- [ ] Find one blocking call inside an `async def` handler and fix it.
- [ ] Generate a TypeScript client from your OpenAPI schema and use it in a frontend.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Nginx → Gunicorn → Uvicorn → FastAPI → services → database
```

## 2. Request Flow

```text
Input       an HTTP request with a JSON body
    ↓
Processing  routed, dependencies resolved, validated by Pydantic, handled
    ↓
Output      a response filtered by response_model, documented automatically
```

## 3. Real-World Usage

**Machine learning services** are FastAPI's strongest niche. The model lives in Python, inference is I/O- and GPU-bound, and the automatically generated OpenAPI schema lets frontend teams integrate without a hand-written contract.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | An async Python framework driven by type hints |
| **Why does it exist?** | Because validation, serialisation and docs should not be three separate truths |
| **Where does it belong?** | Between an ASGI server and your service layer |
| **When should I use it?** | JSON APIs and ML services — not full server-rendered web applications |
