# Uvicorn

> **In one line —** the ASGI server that gives Python an event loop, so one process can hold thousands of waiting requests instead of one.

| | |
|---|---|
| **Category** | Application Server |
| **Architectural Layer** | Between the proxy and the framework |
| **Interface** | ASGI (async) |
| **Related notes** | [Gunicorn](Gunicorn.md) · [Web Servers](Web%20Servers.md) · [FastAPI](FastAPI.md) · [CPython](../03%20-%20Programming%20Languages%20and%20Runtime/CPython.md) · [Event Loop](../05%20-%20Frontend%20Architecture/Event%20Loop.md) |

---

## 1. Short Definition

*What is it?*

Uvicorn is a Python application server implementing **ASGI** — the asynchronous successor to WSGI. It runs an event loop, so a single worker can hold thousands of concurrent requests that are all waiting on I/O.

---

## 2. Purpose

*What is its main purpose?*

To let Python web applications handle high concurrency without one process or thread per request, and to support protocols WSGI cannot express — WebSockets, server-sent events, long-lived streams.

---

## 3. Problem

*What engineering problem does it solve?*

WSGI, Python's original server interface, is **synchronous by design**: one request occupies one worker until it finishes.

```text
WSGI (Gunicorn sync worker)          ASGI (Uvicorn)
worker busy for the whole request    worker free while awaiting I/O
50 ms DB query = 50 ms blocked       50 ms DB query = 50 ms serving others
concurrency = number of workers      concurrency = thousands per worker
no WebSocket support                 WebSockets, SSE, streaming
```

---

## 4. Architecture Position

```text
Internet
    ↓
Nginx                     TLS, static files
    ↓
Gunicorn                  process manager: N workers, restarts, graceful reload
    ↓
UVICORN WORKER            event loop  ← you are here
    ↓
FastAPI / Starlette / Django ASGI
    ↓
Database (with an async driver)
```

> [!IMPORTANT]
> **Uvicorn provides concurrency; Gunicorn provides parallelism.** One Uvicorn worker uses one core because of the [GIL](../03%20-%20Programming%20Languages%20and%20Runtime/CPython.md), so production runs several under a process manager. That is why the standard command exists.

---

## 5. The standard production command

```bash
gunicorn app:app \
  --worker-class uvicorn.workers.UvicornWorker \
  --workers 4 \
  --bind 0.0.0.0:8000
```

```text
Gunicorn: 4 processes → 4 cores
    ↓
Each runs a Uvicorn event loop → thousands of concurrent awaits
    ↓
Total: 4 cores fully used, very high I/O concurrency
```

---

## 6. Processing

```text
Request arrives
    ↓
Event loop assigns it to a coroutine
    ↓
Handler runs until it hits `await`
    ↓
await db.fetch()  → coroutine SUSPENDS, loop serves other requests
    ↓
Database responds → coroutine resumes
    ↓
Response written; the loop moves on
```

---

## 7. The mistake that eliminates the benefit

> [!CAUTION]
> **A synchronous blocking call inside an async handler blocks the entire event loop** — and with it every other concurrent request in that worker.

```python
@app.get("/bad")
async def bad():
    time.sleep(2)              # ✗ blocks the loop; ALL requests wait 2 seconds
    return requests.get(url)   # ✗ blocking HTTP client

@app.get("/good")
async def good():
    await asyncio.sleep(2)     # ✓ suspends only this request
    return await client.get(url)   # ✓ httpx / aiohttp
```

The same applies to **synchronous database drivers**. Using `psycopg2` in an async handler removes the entire reason for running Uvicorn. Use `asyncpg`, `psycopg` 3 in async mode, or SQLAlchemy's async engine.

> [!TIP]
> If a library has no async version, run it in a thread pool (`run_in_executor`, or FastAPI's plain `def` handlers, which it dispatches to a thread automatically).

---

## 8. Real World Example

- **FastAPI's documented deployment** is Uvicorn, usually managed by Gunicorn.
- **Django** supports ASGI now, and running it under Uvicorn is how Django applications gain WebSocket support via Channels.
- **Any Python service handling many slow outbound calls** — aggregating third-party APIs, for instance — benefits enormously from this model.

---

## 9. Communication and Dependencies

- **A process manager** — Gunicorn, or a container orchestrator
- **An ASGI framework** — FastAPI, Starlette, Django, Litestar
- **Async drivers** — `asyncpg`, `redis.asyncio`, `httpx`
- **uvloop and httptools** — optional C implementations that make it considerably faster

---

## 10. Alternatives

```text
Uvicorn        the standard ASGI server; fast, simple
    ↓
Hypercorn      ASGI with HTTP/2 and HTTP/3 support
    ↓
Granian        Rust-based ASGI/WSGI server, very fast
    ↓
Gunicorn sync  WSGI only — fine for CPU-bound or simple apps
    ↓
Daphne         the original ASGI server, from the Django Channels project
```

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Use Uvicorn for I/O-bound Python services: APIs that mostly wait on databases and other services, and anything needing WebSockets.

> [!CAUTION]
> It provides no advantage when your handlers are CPU-bound — the event loop cannot make computation concurrent, and long CPU work blocks every other request. Move that work to a [background worker](../10%20-%20Distributed%20Systems/Background%20Workers.md).

---

## 12. Advantages and Disadvantages

**Advantages**
- Very high concurrency per worker with low memory
- WebSocket and streaming support
- Fast, especially with `uvloop`
- Simple to run and to reason about

**Disadvantages**
- Requires async-aware libraries throughout the stack
- One blocking call ruins everything for that worker
- No process management of its own in production use
- Async code is harder to debug than synchronous code

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Concurrency** | Thousands of simultaneous awaits per worker |
| **Memory** | A few KB per pending request, versus MB per thread |
| **CPU** | One core per worker — scale with processes |
| **Latency** | Excellent while nothing blocks the loop |

---

## 14. Security Considerations

> [!CAUTION]
> Uvicorn is designed to run **behind a reverse proxy**. Exposed directly it lacks the request limits, timeouts and header handling that a hardened edge server provides.

- **Set `--proxy-headers`** and `--forwarded-allow-ips` so the real client IP is trusted only from your proxy
- **Blocking the loop is a denial-of-service vector** — one slow synchronous call stalls all concurrent requests
- **Limit request sizes** at the proxy layer
- Keep it and its dependencies patched

---

## 15. Mental Model

> [!NOTE]
> **Uvicorn is a waiter who never stands at a table waiting for the kitchen.**
>
> They take an order, pass it through, and immediately move to the next table — so one waiter serves a hundred. But the moment they sit down to chop vegetables themselves, every table waits. That is a blocking call in an async handler.

---

## 16. Mini Architecture Diagram

```text
Nginx
    ↓
Gunicorn (process manager)
    ↓
┌──────────┬──────────┬──────────┬──────────┐
│ Uvicorn  │ Uvicorn  │ Uvicorn  │ Uvicorn  │  ← one per core
│ worker 1 │ worker 2 │ worker 3 │ worker 4 │
│ (loop)   │ (loop)   │ (loop)   │ (loop)   │
└──────────┴──────────┴──────────┴──────────┘
    ↓
FastAPI application
    ↓
async database driver
```

---

## 17. Complete Request Flow

```text
Request → Nginx → Gunicorn → one Uvicorn worker
    ↓
Event loop schedules the coroutine
    ↓
FastAPI: routing → validation → dependency injection
    ↓
await db.fetch()  → coroutine suspends
    ↓
Worker serves other requests during the wait     ← the entire point
    ↓
Database responds → coroutine resumes
    ↓
Response serialised and returned
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> Uvicorn gives one Python process an event loop and enormous I/O concurrency — but a single blocking call inside a handler removes that benefit entirely.

---

## 19. Common Mistakes

- **Blocking calls in async handlers** — `time.sleep`, `requests`, synchronous DB drivers
- **Running Uvicorn alone in production** without a process manager or orchestrator
- **Using it for CPU-bound work** and expecting concurrency to help
- **Forgetting `--proxy-headers`**, so the app logs the proxy's IP
- **Mixing sync and async database sessions** in the same codebase
- **Exposing it directly** to the internet

---

## 20. Open Source Technologies

- **Uvicorn**, **Hypercorn**, **Granian**, **Daphne** — ASGI servers
- **uvloop**, **httptools** — C speedups
- **FastAPI**, **Starlette**, **Litestar** — ASGI frameworks
- **asyncpg**, **httpx**, **redis.asyncio** — async clients

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Add a `time.sleep(2)` to an async endpoint and observe what happens to concurrent requests.
- [ ] Audit your handlers for synchronous I/O calls and list what needs an async replacement.
- [ ] Verify your worker count matches your available cores.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Nginx → Gunicorn → Uvicorn workers → FastAPI → async DB driver
```

## 2. Request Flow

```text
Input       an HTTP request handed over by the process manager
    ↓
Processing  scheduled as a coroutine; suspended at every await
    ↓
Output      a response, with the worker serving others throughout the wait
```

## 3. Real-World Usage

**FastAPI's official deployment guide** recommends Uvicorn workers under Gunicorn. That combination is now the default shape of production Python APIs, precisely because it pairs event-loop concurrency with process-level parallelism.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | An ASGI server running an event loop for Python |
| **Why does it exist?** | Because WSGI blocks a worker for the whole request |
| **Where does it belong?** | Between a process manager and an async framework |
| **When should I use it?** | I/O-bound Python services and WebSockets — never with blocking calls |
