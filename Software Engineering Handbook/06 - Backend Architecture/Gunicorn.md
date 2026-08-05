# Gunicorn

> **In one line —** a process manager for Python web applications: it forks workers, restarts them when they die, and is how Python actually uses more than one CPU core.

| | |
|---|---|
| **Full name** | Green Unicorn |
| **Category** | Application Server / Process Manager |
| **Architectural Layer** | Between the proxy and the framework |
| **Interface** | WSGI (and ASGI via Uvicorn workers) |
| **Related notes** | [Uvicorn](Uvicorn.md) · [Web Servers](Web%20Servers.md) · [CPython](../03%20-%20Programming%20Languages%20and%20Runtime/CPython.md) · [Process](../02%20-%20Computer%20Science%20Fundamentals/Process.md) · [Runtime Explained](Runtime%20Explained.md) |

---

## 1. Short Definition

*What is it?*

Gunicorn is a pre-fork worker model server for Python. A master process starts several worker processes, distributes connections to them, restarts them if they crash, and reloads them gracefully on deploy.

---

## 2. Purpose

*What is its main purpose?*

To turn a single-threaded Python application into one that uses every core on the machine, and to keep it running when individual workers fail.

---

## 3. Problem

*What engineering problem does it solve?*

[CPython's GIL](../03%20-%20Programming%20Languages%20and%20Runtime/CPython.md) means one process executes Python bytecode on one core, no matter how many threads you create.

```text
8-core server, one Python process
    ↓
1 core used, 7 idle
    ↓
Gunicorn: 8 worker PROCESSES
    ↓
8 cores used
```

Processes are the only way to get true CPU parallelism in CPython, and Gunicorn is the standard way to manage them.

---

## 4. Architecture Position

```text
Nginx
    ↓
GUNICORN MASTER          ← binds the socket, manages workers
    ├── worker 1
    ├── worker 2
    ├── worker 3
    └── worker 4
            ↓
    Your Django / Flask / FastAPI application
            ↓
    Database
```

The master never handles requests. It owns the listening socket and supervises children.

---

## 5. Worker types — the decision that matters

| Worker class | Model | Use for |
|---|---|---|
| **sync** (default) | One request per worker at a time | CPU-bound, simple apps |
| **gthread** | Threads within each worker | Mild I/O concurrency |
| **gevent** / **eventlet** | Green threads, monkey-patched | High I/O concurrency, legacy sync code |
| **uvicorn.workers.UvicornWorker** | Full async event loop | FastAPI, Starlette, async Django |

```bash
# Synchronous WSGI app (Django, Flask)
gunicorn app:app --workers 5

# Async ASGI app (FastAPI)
gunicorn app:app --worker-class uvicorn.workers.UvicornWorker --workers 4
```

---

## 6. How many workers

```text
Sync workers, I/O-bound:   (2 × cores) + 1        ← the documented rule of thumb
Sync workers, CPU-bound:   cores
Uvicorn workers (async):   cores                  ← concurrency comes from the loop
```

> [!CAUTION]
> **Memory is usually the real limit.** Each worker carries its own copy of the interpreter and every imported library — commonly 100–300 MB for a Django or ML-adjacent application. Four workers at 250 MB is a gigabyte before a single request arrives, and a 512 MB container will be OOM-killed.

---

## 7. Processing

```text
Master binds the port and forks N workers
    ↓
Kernel distributes incoming connections among the workers
    ↓
Each worker handles requests independently, in its own memory
    ↓
A worker crashes  → master forks a replacement; other requests unaffected
    ↓
A worker exceeds --timeout → master kills and replaces it
    ↓
SIGHUP → workers reloaded one at a time, with no dropped connections
```

> [!IMPORTANT]
> **Workers share nothing.** An in-memory cache, a counter or a global variable exists separately in each worker. State that must be shared belongs in [Redis](../08%20-%20Databases%20and%20Data/Redis.md) or the database — this surprises people constantly.

---

## 8. Real World Example

- **The standard Django production deployment** is Nginx → Gunicorn → Django.
- **The standard FastAPI deployment** is Gunicorn managing Uvicorn workers.
- **Rolling reloads** via SIGHUP allow zero-downtime deploys on a single machine.

---

## 9. Communication and Dependencies

- **A reverse proxy** in front — Gunicorn is explicitly not designed to face the internet
- **A WSGI or ASGI application** behind
- **The operating system** for `fork()`; copy-on-write makes forking cheap
- **A supervisor** — systemd, Docker or Kubernetes — to keep the master alive

---

## 10. Alternatives

```text
Gunicorn      the standard; simple, reliable, well documented
    ↓
uWSGI         more features, far more configuration complexity
    ↓
Granian       Rust-based, faster, newer
    ↓
Uvicorn alone in a container, with the orchestrator providing replicas
    ↓
Waitress      pure Python, works on Windows
```

> [!TIP]
> In Kubernetes, running **one Uvicorn process per container** and letting the orchestrator provide replicas is often simpler than Gunicorn inside the container. Two layers of process management can obscure what is actually failing.

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Use Gunicorn for any Python web application deployed to a VM or a container where you want multiple workers, automatic restarts and graceful reloads.

> [!CAUTION]
> Do not expose it directly to the internet — it has no TLS, weak protection against slow clients, and no static file handling. And do not run it without measuring memory per worker first.

---

## 12. Advantages and Disadvantages

**Advantages**
- Uses all cores despite the GIL
- Worker crashes are contained and automatically recovered
- Graceful reload gives zero-downtime deploys
- Simple, stable, minimal configuration
- Supports both WSGI and ASGI through worker classes

**Disadvantages**
- Memory multiplies by worker count
- No shared state between workers
- Not safe to expose directly
- Timeout handling is blunt — the worker is killed, not the request
- Choosing the wrong worker class silently costs you most of the performance

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Throughput** | Scales roughly linearly with cores for CPU work |
| **Memory** | Workers × per-worker footprint — the usual constraint |
| **Startup** | Each worker re-imports everything |
| **Resilience** | One bad request kills one worker, not the service |

---

## 14. Security Considerations

> [!CAUTION]
> Gunicorn behind no proxy is vulnerable to slow-client attacks (Slowloris): a handful of connections trickling bytes can occupy every sync worker and take the service down. Nginx buffering is the standard mitigation.

- **Always run behind a reverse proxy**
- **Set `--forwarded-allow-ips`** so `X-Forwarded-For` is trusted only from your proxy
- **Set `--timeout`** so a hung request cannot occupy a worker indefinitely
- **Run as a non-root user**
- **Use `--max-requests` with jitter** to recycle workers and limit the impact of memory leaks

---

## 15. Mental Model

> [!NOTE]
> **Gunicorn is a shift manager with several identical staff.**
>
> The manager takes no customers. They open the shop, put four people on the counter, and replace anyone who collapses. Each staff member has their own till and their own notes — none of them can see what the others wrote down, which is why shared information has to go on the noticeboard (Redis).

---

## 16. Mini Architecture Diagram

```text
Nginx
    ↓
Gunicorn master  (owns the socket, supervises)
    ├─ worker 1  → own memory, own DB pool
    ├─ worker 2  → own memory, own DB pool
    ├─ worker 3  → own memory, own DB pool
    └─ worker 4  → own memory, own DB pool
                       ↓
              Redis (shared state)  ·  PostgreSQL
```

---

## 17. Complete Request Flow

```text
Request → Nginx → Gunicorn's listening socket
    ↓
Kernel hands the connection to an available worker
    ↓
Worker runs the WSGI/ASGI application
    ↓
Application queries the database via THAT WORKER'S pool
    ↓
Response returned through Nginx
    ↓
Worker exceeded --timeout?  → master kills and replaces it
    ↓
Deploy: SIGHUP → workers replaced one by one, no connections dropped
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> Gunicorn gives Python parallelism through processes because the GIL prevents it through threads — and because workers share nothing, all shared state must live outside them.

---

## 19. Common Mistakes

- **Too many workers for the available memory** → OOM kills
- **Wrong worker class** — sync workers for an async framework, losing all concurrency
- **Expecting workers to share memory** — in-process caches and counters do not propagate
- **Exposing it directly** to the internet
- **No `--timeout`**, so one hung request holds a worker forever
- **Sizing database connection pools per worker** without multiplying by worker and replica count

---

## 20. Open Source Technologies

- **Gunicorn** — the server
- **Uvicorn**, **uvloop** — async worker class
- **uWSGI**, **Granian**, **Waitress** — alternatives
- **supervisord**, **systemd** — keeping the master alive outside containers

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Measure the memory of one worker under load, multiply by your worker count, and compare with your container limit.
- [ ] Calculate total database connections: workers × pool size × replicas.
- [ ] Confirm your worker class matches your framework (sync vs ASGI).

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Nginx → Gunicorn master → N workers → application → database
```

## 2. Request Flow

```text
Input       connections on a shared listening socket
    ↓
Processing  distributed to worker processes, each isolated
    ↓
Output      responses, with crashed workers replaced automatically
```

## 3. Real-World Usage

Almost every production Django and FastAPI deployment in existence runs Gunicorn. It is the piece that turns CPython's single-core execution model into something that can use a whole server.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A pre-fork process manager for Python web applications |
| **Why does it exist?** | Because the GIL means only processes give CPU parallelism |
| **Where does it belong?** | Between a reverse proxy and your framework |
| **When should I use it?** | Any Python web app in production — sized by memory, not by rule of thumb alone |
