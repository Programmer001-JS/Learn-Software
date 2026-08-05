# Django

> **In one line —** a Python framework that hands you an ORM, an admin interface, authentication and migrations on the first day, so you build the product instead of the plumbing.

| | |
|---|---|
| **Category** | Web Framework |
| **Architectural Layer** | Application |
| **Language** | Python |
| **Philosophy** | Batteries included · "Don't repeat yourself" |
| **Related notes** | [Backend Frameworks](Backend%20Frameworks.md) · [FastAPI](FastAPI.md) · [ORM](../08%20-%20Databases%20and%20Data/ORM.md) · [Gunicorn](Gunicorn.md) · [CPython](../03%20-%20Programming%20Languages%20and%20Runtime/CPython.md) |

---

## 1. Short Definition

*What is it?*

Django is a batteries-included Python web framework. It ships an ORM, database migrations, an authentication system, an automatically generated admin interface, a template engine, forms and a large set of security defaults.

---

## 2. Purpose

*What is its main purpose?*

To get a complete, secure, database-backed web application into production quickly — with conventions already decided so a team does not spend its first month choosing libraries.

---

## 3. Problem

*What engineering problem does it solve?*

Every data-backed application needs the same twenty things. Assembling them from separate packages costs weeks and produces a different result in every project.

```text
MICRO FRAMEWORK                  DJANGO
choose an ORM                    included
choose migrations                included
choose auth                      included
choose an admin                  included and generated from your models
choose a template engine         included
wire it together                 already wired
```

---

## 4. Architecture Position

```text
Nginx
    ↓
Gunicorn (WSGI) or Uvicorn (ASGI)
    ↓
┌──────────── DJANGO ────────────┐
│  Middleware                    │
│  URL routing                   │
│  View  (function or class)     │
│  Forms / DRF serialisers       │
│  Models (ORM)                  │
└───────────────┬────────────────┘
                ↓
            PostgreSQL
```

---

## 5. The MTV structure

```text
MODEL      a Python class that defines a database table
VIEW       the function handling a request  (called a "controller" elsewhere)
TEMPLATE   the HTML rendering layer
```

```python
class Order(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    total = models.DecimalField(max_digits=10, decimal_places=2)
    created = models.DateTimeField(auto_now_add=True)
```

From that one class you get: a database table, migrations, an admin CRUD interface, form validation and a query API.

---

## 6. The admin — the feature that sells it

`django-admin` generates a working CRUD interface for every model, with search, filtering and permissions, from your model definitions alone.

> [!TIP]
> For internal tools, back-office systems and early-stage products, the admin frequently removes weeks of work. It is the single most cited reason teams choose Django.

> [!CAUTION]
> It is an **internal tool**, not a customer-facing interface, and it grants powerful direct database access. Restrict it by IP or VPN, and never assume staff permissions are fine-grained enough for external users.

---

## 7. Django REST Framework

Django itself is oriented toward server-rendered HTML. For JSON APIs, **DRF** adds serialisers, viewsets, authentication classes and permissions — and is effectively part of the standard stack.

```text
Django + templates     traditional server-rendered web application
Django + DRF           JSON API for a React/mobile frontend
Django + Channels      WebSockets and async
```

---

## 8. Real World Example

- **Instagram** runs one of the largest Django deployments in the world, scaled horizontally with many worker processes.
- **Mozilla, Disqus and Pinterest** have all run substantial Django systems.
- **Internal tools everywhere** — the admin makes Django disproportionately common for back-office software.

---

## 9. Communication and Dependencies

- **[Gunicorn](Gunicorn.md)** (WSGI) or Uvicorn (ASGI) as the application server
- **PostgreSQL** — the best-supported database by a wide margin
- **Celery + Redis** for [background work](../10%20-%20Distributed%20Systems/Background%20Workers.md)
- **DRF** for APIs

---

## 10. Alternatives

```text
Django       batteries included; ORM, admin, auth, migrations
    ↓
FastAPI      async, API-only, type-driven; you assemble the rest
    ↓
Flask        minimal; choose everything yourself
    ↓
Laravel      the PHP equivalent in philosophy and ergonomics
    ↓
Rails        the original batteries-included framework
```

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Use Django for database-heavy applications with substantial CRUD, where an admin interface is valuable and delivery speed matters — SaaS products, internal systems, content platforms, marketplaces.

> [!CAUTION]
> - **High-concurrency async APIs** are a better fit for [FastAPI](FastAPI.md); Django's async support exists but the ORM is still largely synchronous.
> - **A small three-endpoint service** does not need the machinery.
> - **Non-relational data models** fight the ORM rather than benefit from it.

---

## 12. Advantages and Disadvantages

**Advantages**
- Extremely fast from zero to a working product
- Excellent, safe-by-default ORM with real migrations
- The admin interface is genuinely unmatched
- Strong security defaults — CSRF, XSS escaping, SQL injection protection
- Enormous, stable ecosystem and outstanding documentation

**Disadvantages**
- Opinionated — fighting its conventions is painful
- Monolithic by design; splitting it later is real work
- The ORM makes inefficient queries easy to write accidentally
- Async support is partial; the ORM remains the bottleneck
- Heavier than a micro framework for simple services

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Throughput** | Adequate; scale horizontally with worker processes |
| **Concurrency** | WSGI: one request per worker. ASGI: better, but the ORM limits it |
| **Memory** | 100–300 MB per worker — multiply by worker count |
| **The real risk** | **N+1 queries** from the ORM, not framework overhead |

> [!IMPORTANT]
> Django performance problems are almost always **database query problems**. `select_related` and `prefetch_related` fix more slow pages than any amount of code tuning. Install `django-debug-toolbar` and look at the query count before optimising anything else.

---

## 14. Security Considerations

Django has the best security defaults of any framework in this handbook — which means most incidents come from turning them off.

> [!CAUTION]
> - **`DEBUG = True` in production** exposes settings, environment variables and full stack traces. It is a complete compromise and it happens regularly.
> - **`|safe` in templates** disables XSS escaping.
> - **`csrf_exempt`** added during debugging and never removed.
> - **Raw SQL with string formatting** bypasses ORM parameterisation.

- **`ALLOWED_HOSTS`** must be set correctly — it prevents host-header attacks
- **`SECRET_KEY` must never be in source control**
- **Object-level permissions are yours** — Django checks model permissions, not "does this order belong to this user"
- Run `python manage.py check --deploy` before going live

---

## 15. Mental Model

> [!NOTE]
> **Django is a furnished flat; FastAPI is an empty one.**
>
> The furnished flat lets you move in tonight, with furniture someone thought carefully about. If you want a different kitchen layout, you are rearranging someone else's decisions. The empty flat needs weeks of work before it is liveable — and then it is exactly what you wanted.

---

## 16. Mini Architecture Diagram

```text
Request
    ↓
Middleware  (security, session, CSRF, auth)
    ↓
URL router
    ↓
View  →  Form / DRF serialiser (validation)
    ↓
Model / ORM
    ↓
PostgreSQL
    ↓
Template or JSON response
```

---

## 17. Complete Request Flow

```text
Request → Nginx → Gunicorn → one Django worker
    ↓
Middleware chain: security headers, session, CSRF, authentication
    ↓
URL resolved to a view
    ↓
Permission check
    ↓
Form or serialiser validates the input
    ↓
ORM query — watch the query count here
    ↓
Template rendered (auto-escaped) or JSON serialised
    ↓
Response through the middleware chain in reverse
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> Django trades flexibility for a complete, secure, conventional stack on day one — and its performance ceiling is set by ORM query patterns far more than by the framework itself.

---

## 19. Common Mistakes

- **`DEBUG = True` in production**
- **N+1 queries** — the single most common Django performance problem
- **`|safe` and `csrf_exempt`** left in after debugging
- **Business logic in views**, making it untestable
- **Fat models with no service layer** in large applications
- **Secrets committed** to the repository
- **Expecting async performance** from a mostly synchronous ORM

---

## 20. Open Source Technologies

- **Django**, **Django REST Framework**
- **django-debug-toolbar** — see every query a page runs
- **Celery** — background tasks
- **django-environ** — configuration from the environment
- **pytest-django**, **factory_boy** — testing
- **Wagtail** — a CMS built on Django

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Install django-debug-toolbar and find your page with the highest query count.
- [ ] Fix one N+1 with `select_related` or `prefetch_related` and measure the difference.
- [ ] Run `manage.py check --deploy` and address every warning.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Nginx → Gunicorn → Django (middleware → view → ORM) → PostgreSQL
```

## 2. Request Flow

```text
Input       an HTTP request
    ↓
Processing  middleware → routing → view → validation → ORM
    ↓
Output      rendered HTML or JSON, with security defaults applied throughout
```

## 3. Real-World Usage

**Instagram** scaled Django to hundreds of millions of users by running very many worker processes and caching aggressively. It is the standing counter-argument to the idea that a batteries-included framework cannot scale.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A batteries-included Python framework with ORM, admin and auth |
| **Why does it exist?** | Because every database-backed app needs the same twenty components |
| **Where does it belong?** | Between a WSGI/ASGI server and a relational database |
| **When should I use it?** | CRUD-heavy products where delivery speed and an admin matter |
