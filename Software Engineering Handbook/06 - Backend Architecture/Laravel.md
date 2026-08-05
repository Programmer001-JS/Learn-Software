# Laravel

> **In one line —** the PHP framework that made PHP pleasant again: expressive syntax, batteries included, and the best developer ergonomics of any framework in this section.

| | |
|---|---|
| **Category** | Web Framework |
| **Architectural Layer** | Application |
| **Language** | PHP |
| **Philosophy** | Batteries included · developer happiness |
| **Related notes** | [Backend Frameworks](Backend%20Frameworks.md) · [Django](Django.md) · [ORM](../08%20-%20Databases%20and%20Data/ORM.md) · [Background Workers](../10%20-%20Distributed%20Systems/Background%20Workers.md) |

---

## 1. Short Definition

*What is it?*

Laravel is a batteries-included PHP framework. It ships an ORM (Eloquent), migrations, authentication, queues, task scheduling, mail, caching, real-time broadcasting and a templating engine, with an emphasis on readable, expressive code.

---

## 2. Purpose

*What is its main purpose?*

To build complete web applications quickly, with conventions decided and a toolchain that covers nearly everything a typical product needs without adding third-party packages.

---

## 3. Problem

*What engineering problem does it solve?*

PHP's reputation was earned honestly: inconsistent standard library, no structure, and code that grew into unmaintainable page scripts. Laravel imposed modern architecture — dependency injection, a service container, MVC — on a language that runs on essentially every host on earth.

---

## 4. Architecture Position

```text
Nginx
    ↓
PHP-FPM              ← process pool; one request per process at a time
    ↓
┌──────────── LARAVEL ────────────┐
│  Middleware                     │
│  Router                         │
│  Controller                     │
│  Form Request  (validation)     │
│  Service / Action               │
│  Eloquent ORM                   │
└──────────────┬──────────────────┘
               ↓
           MySQL / PostgreSQL
```

> [!IMPORTANT]
> PHP's execution model is unusual and worth understanding: **each request starts from a clean slate.** The framework boots, handles one request, and shuts down. There is no long-lived process holding state between requests — which makes memory leaks nearly impossible and makes per-request boot cost the main performance concern.

---

## 5. What it includes

| Feature | Provided |
|---|---|
| **Eloquent ORM** | Active Record with relationships and eager loading |
| **Migrations** | Version-controlled schema changes |
| **Authentication** | Breeze, Jetstream, Sanctum, Passport |
| **Queues** | Redis, SQS, database drivers, with a built-in worker |
| **Scheduler** | Cron expressions defined in code |
| **Blade** | Templating with auto-escaping |
| **Artisan** | A CLI for generation, migration and maintenance |
| **Broadcasting** | WebSockets via Pusher, Reverb or Soketi |

---

## 6. The developer experience

```php
$orders = Order::with('items')            // eager load — avoids N+1
    ->where('status', 'pending')
    ->latest()
    ->paginate(20);
```

Laravel's expressiveness is its actual product. Pagination, eager loading, validation and authorisation are each a line rather than a subsystem.

---

## 7. Real World Example

- **Small and mid-sized SaaS products** are Laravel's strongest niche — the full stack in one framework with a small team.
- **Agency and client work** — fast delivery with predictable structure.
- **Laravel Forge and Vapor** made deployment nearly as opinionated as the framework.

---

## 8. Communication and Dependencies

- **PHP-FPM** behind Nginx
- **MySQL or PostgreSQL**
- **Redis** for cache, sessions and queues
- **Composer** for packages
- **Laravel Horizon** for queue monitoring

---

## 9. Alternatives

```text
Laravel      batteries included, best PHP ergonomics
    ↓
Symfony      more modular and enterprise-oriented; Laravel uses its components
    ↓
Django       the Python equivalent in philosophy
    ↓
Rails        the original of this family
    ↓
Slim         minimal PHP, when you want no framework opinions
```

---

## 10. When To Use / When NOT To Use

> [!TIP]
> Use Laravel for complete web applications — CRUD-heavy products, SaaS, client work — especially with a small team where speed of delivery matters most and PHP hosting is abundant and cheap.

> [!CAUTION]
> - **High-concurrency real-time systems** fit poorly with the request-per-process model; Node or Go handle persistent connections far better.
> - **CPU-heavy or data-science work** — the ecosystem is not there.
> - **Very high throughput** requires Octane (persistent workers), which changes the execution model and invalidates the "clean slate per request" assumption your code may rely on.

---

## 11. Advantages and Disadvantages

**Advantages**
- Extremely productive; most features are one line
- Complete stack — queues, scheduling, mail, auth, broadcasting included
- Excellent documentation and a large, active community
- Cheap, universal hosting
- No memory leaks by construction, thanks to per-request lifecycles

**Disadvantages**
- Per-request boot cost unless you run Octane
- Eloquent's Active Record couples models to the database, which strains in complex domains
- "Facades" and magic methods hide what is actually happening
- Poor fit for persistent connections and high concurrency
- PHP's ecosystem is weak outside web development

---

## 12. Performance Impact

| Aspect | Impact |
|---|---|
| **Per request** | Framework boot on every request — mitigated by opcache, removed by Octane |
| **Concurrency** | One request per PHP-FPM process |
| **Memory** | Modest per process, freed after every request |
| **The real risk** | **N+1 queries** from Eloquent relationships |

> [!TIP]
> As with Django, Laravel performance problems are usually query problems. `with()` for eager loading and Laravel Debugbar for query counts solve most of them.

---

## 13. Security Considerations

Laravel has strong defaults, and the usual failures are disabling them.

> [!CAUTION]
> - **`APP_DEBUG=true` in production** exposes environment variables, including database credentials and API keys, on any error page. This has caused real breaches.
> - **`{!! !!}` in Blade** disables escaping — an XSS vector.
> - **`DB::raw()` with user input** bypasses parameterisation.
> - **Mass assignment** — always define `$fillable`, or a request can set fields like `is_admin`.

- **Policies and Gates** handle object-level authorisation; use them rather than role checks alone
- **`.env` must never be committed**, and must not be readable from the web root
- Keep Composer dependencies patched

---

## 14. Mental Model

> [!NOTE]
> **Laravel is a well-equipped workshop where every tool is already on the wall, labelled and within reach.**
>
> You spend your time building rather than looking for a screwdriver. The trade is that the workshop is laid out someone else's way — and if you want the bench somewhere different, you are moving a lot of shelves.

---

## 15. Mini Architecture Diagram

```text
Request
    ↓
Nginx → PHP-FPM
    ↓
Middleware (auth, CSRF, throttle)
    ↓
Router → Controller
    ↓
Form Request (validation) → Policy (authorisation)
    ↓
Service / Action
    ↓
Eloquent → database
    ↓
Blade view or JSON resource
```

---

## 16. Complete Request Flow

```text
Request arrives at Nginx → passed to a PHP-FPM process
    ↓
Laravel BOOTS  (fresh application instance)
    ↓
Middleware: session, CSRF, authentication, rate limiting
    ↓
Route matched → controller method
    ↓
Form Request validates → 422 with field errors if invalid
    ↓
Policy authorises the specific object → 403 if refused
    ↓
Eloquent query (eager-loaded to avoid N+1)
    ↓
JSON resource or Blade template (auto-escaped)
    ↓
Response sent; the process is reset for the next request
```

---

## 17. Key Takeaway

> [!IMPORTANT]
> Laravel gives you a complete application stack with excellent ergonomics, on a runtime that starts fresh for every request — productive and safe by construction, and a poor fit for persistent-connection workloads.

---

## 18. Common Mistakes

- **`APP_DEBUG=true` in production**
- **N+1 queries** from unloaded relationships
- **`{!! !!}`** with user content
- **No `$fillable`**, allowing mass assignment
- **Business logic in controllers** rather than actions or services
- **`.env` committed or web-accessible**
- **Skipping policies** and checking only roles

---

## 19. Open Source Technologies

- **Laravel**, **Symfony** — frameworks
- **Laravel Octane** — persistent workers via Swoole or RoadRunner
- **Horizon** — queue dashboard
- **Laravel Debugbar** — query and performance inspection
- **Pest**, **PHPUnit** — testing
- **Composer** — dependency management

---

## 20. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 21. Workbook Exercise

- [ ] Confirm `APP_DEBUG` is false in production and that `.env` is not web-accessible.
- [ ] Find one N+1 query with Debugbar and fix it with `with()`.
- [ ] Take one authorisation check based on a role and replace it with a Policy that checks the object.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Nginx → PHP-FPM → Laravel (middleware → controller → Eloquent) → MySQL
```

## 2. Request Flow

```text
Input       an HTTP request
    ↓
Processing  framework boots, middleware runs, validation and authorisation, ORM
    ↓
Output      HTML or JSON, with the process reset afterwards
```

## 3. Real-World Usage

Laravel dominates **small-to-medium SaaS and agency work**, where one small team must deliver a complete product. Queues, scheduling, mail and auth being included means fewer decisions and fewer integrations to maintain.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A batteries-included PHP framework with strong ergonomics |
| **Why does it exist?** | To bring modern architecture to a language that lacked it |
| **Where does it belong?** | Between PHP-FPM and a relational database |
| **When should I use it?** | Complete web applications built quickly by a small team |
