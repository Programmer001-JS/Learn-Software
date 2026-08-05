# Redis

> **In one line —** an in-memory key-value store that sits in front of your database and answers repeated questions in microseconds.

| | |
|---|---|
| **Category** | Infrastructure Component |
| **Architectural Layer** | Data / Caching |
| **Written in** | C |
| **Related notes** | [Cache](Cache.md) · [Session Store](Session%20Store.md) · [Pub Sub](Pub%20Sub.md) · [Distributed Lock](Distributed%20Lock.md) · [PostgreSQL](PostgreSQL.md) · [Caching](../10%20-%20Distributed%20Systems/Caching.md) |

---

## 1. Short Definition

*What is it?*

Redis (**RE**mote **DI**ctionary **S**erver) is an **in-memory key-value data store**. It keeps all of its data in RAM instead of on disk, which is why a read takes microseconds instead of milliseconds. Beyond simple caching it also serves as a session store, message broker, rate limiter and distributed lock.

---

## 2. Purpose

*What is its main purpose?*

To hold **fast, shared, temporary state** that every application server can reach. Its job is to absorb repeated reads so the primary database only handles work that genuinely needs it.

---

## 3. Problem

*What engineering problem does it solve?*

A database read is expensive — disk I/O, query parsing, joins, locking. If 10,000 users open the same product page, PostgreSQL runs the same query 10,000 times and returns the same answer every time.

Without Redis you are left with two bad options:

- **Hammer the database** until it becomes the bottleneck for the entire system.
- **Cache in each server's local memory**, which breaks the moment you run more than one server, because each one then holds a different version of the truth.

---

## 4. Architecture Position

*Where is it located inside the architecture?*

Redis sits **between the application layer and the database**. It is not the source of truth — it is a fast layer standing in front of the source of truth.

```text
Browser
    ↓
Frontend  (React / Next.js)
    ↓
API / Application Layer  (FastAPI, Express, Spring Boot)
    ↓
Redis                    ← in-memory layer: cache, sessions, queues
    ↓
PostgreSQL               ← source of truth, stored on disk
```

> [!IMPORTANT]
> Redis is always reached **by the backend**, never directly by the browser.

---

## 5. Real World Example

*Where is it used in real-world software?*

- **Twitter / X** — caches timelines so a feed is not rebuilt from the database on every refresh.
- **GitHub** — session storage and background job queues.
- **Stack Overflow** — caches rendered question pages, which are read far more often than they are written.
- **Uber** — holds fast-changing dispatch and driver-location state that would be far too write-heavy for a relational database.

The common pattern in all four: **read-heavy traffic where the same answer is requested many times.**

---

## 6. Input

*What does it receive as input?*

Commands sent over TCP using the RESP protocol by a client library inside your backend. A command consists of a key, an operation, and optionally a value and a TTL (time to live):

```text
SET  user:42:profile  '{"name":"Ivan"}'  EX 300
GET  user:42:profile
DEL  user:42:profile
```

Values are not limited to strings — Redis also stores **lists, hashes, sets, sorted sets and streams**.

---

## 7. Processing

*What happens inside it?*

Redis looks the key up in a hash table held entirely in RAM and executes the command. Commands run **one at a time on a single thread**, so every command is atomic — there are no locks and no race conditions between two commands.

In the background it expires keys whose TTL has passed, and optionally writes a snapshot (RDB) or an append-only log (AOF) to disk.

---

## 8. Output

*What does it return?*

The stored value, or `nil` if the key does not exist or has already expired — that case is called a **cache miss**. Most key operations are O(1), so the response typically arrives in well under a millisecond.

---

## 9. Internal Idea

*How does it work internally?*

Picture one enormous hash table (dictionary) living in RAM, driven by a single-threaded event loop that processes commands in the order they arrive.

The speed does not come from clever parallelism. It comes from **avoiding the disk entirely**, and from being simple enough that no locking is ever needed. Persistence, when enabled, happens in the background and is a safety net — not the main path.

---

## 10. Communication

*Which components does it communicate with?*

- **API / application servers** — the main clients
- **Background workers** (Celery, BullMQ) — which use Redis as their job queue
- **Redis replicas** — for replication and read scaling
- **The database** — indirectly; on a miss the application reads PostgreSQL and writes the result into Redis

---

## 11. Dependencies

*What does it depend on?*

- A running Redis server process
- Enough **RAM** to hold the working set
- A network path from the backend
- A client library (`redis-py`, `ioredis`, `Lettuce`, …)

> [!WARNING]
> Redis without a source of truth behind it is not a cache — it is a fragile database.

---

## 12. Alternatives

*What alternative solutions exist?*

```text
Redis               full-featured: cache, queue, pub/sub, locks
    ↓
Valkey              open-source fork of Redis, same commands
    ↓
Memcached           simpler, cache only, no data structures, no persistence
    ↓
Local memory cache  fastest, but private to one server — breaks when you scale out
```

Also worth knowing: **KeyDB** and **Dragonfly** (multi-threaded, Redis-compatible servers), and **DynamoDB DAX** on AWS.

---

## 13. When To Use

*In which situations should this be used?*

> [!TIP]
> Use Redis when the same data is read many times **and** being a few seconds out of date is acceptable.

---

## 14. When NOT To Use

*When should it NOT be used?*

> [!CAUTION]
> Redis is the wrong tool when:
> - The data must never be lost — use PostgreSQL, not Redis.
> - The dataset is larger than the available RAM.
> - Every request is unique (one-off per-user queries), so the cache would never be hit.
> - A single application server would do — a local in-process cache is simpler and has no network hop.

---

## 15. Advantages

- **Sub-millisecond** reads and writes
- Dramatically **reduces load** on the primary database
- **Shared** by all application instances, so it scales horizontally
- **Rich data structures** — lists, sets, sorted sets, streams — not just strings
- **Built-in TTL**, so stale data cleans itself up automatically

---

## 16. Disadvantages

- **RAM is expensive** compared to disk, so the dataset size is limited by cost
- **Data can be lost on crash** unless persistence is carefully configured
- Introduces **cache invalidation**, one of the genuinely hard problems in software
- **One more service** to deploy, monitor, secure and back up
- **Single-threaded** — one slow command such as `KEYS *` blocks everything else

---

## 17. Performance Impact

*How does it affect application performance?*

| Resource | Impact |
|---|---|
| **Speed** | Huge win — a cached read is roughly 10–100× faster than a database query |
| **Memory** | Everything lives in RAM, so memory becomes the limiting resource and the main cost |
| **CPU** | Very low; Redis mostly uses a single core |
| **GPU** | Not used |
| **Network** | Adds one extra hop, but the payload is small and the hop is far cheaper than a disk read |

---

## 18. Security Considerations

*Does it introduce any security concerns?*

> [!CAUTION]
> Redis was designed to run inside a **trusted network** and historically shipped **without authentication**. An exposed Redis instance on the public internet is a well-known and frequently exploited breach.

- Bind it to a **private network**, never to a public interface
- Set `requirepass` or configure **ACL users**
- Enable **TLS** in production
- Remember that cached data inherits its own sensitivity — cached profiles may contain personal data
- If sessions live in Redis, **compromising Redis means compromising every logged-in user**

---

## 19. Mental Model

*What real-life object can it be compared to?*

> [!NOTE]
> **Redis = the refrigerator in your kitchen.**
> **The database = the warehouse across town.**

You keep what you use often in the fridge because reaching it takes seconds. The warehouse holds everything permanently, but going there takes an hour. The fridge is small, its contents expire, and losing it is annoying but not catastrophic — you can always restock from the warehouse.

---

## 20. Mini Architecture Diagram

```text
User
    ↓
Browser
    ↓
Frontend
    ↓
API
    ↓
Redis  ──► HIT ──► return immediately
    ↓
   MISS
    ↓
Database
```

---

## 21. Complete Request Flow

**Cache HIT** — the common case:

```text
Request  "give me product 42"
    ↓
API checks Redis:  GET product:42
    ↓
Cache HIT — value found
    ↓
Response          (the database is never touched)
```

**Cache MISS** — the first request, or after the TTL expired:

```text
Request  "give me product 42"
    ↓
API checks Redis:  GET product:42
    ↓
Cache MISS — nil returned
    ↓
Query PostgreSQL
    ↓
Write result back:  SET product:42 ... EX 300
    ↓
Response
```

Every later request for the next 5 minutes takes the short path.

---

## 22. Key Takeaway

> [!IMPORTANT]
> Redis exists to reduce database reads and improve application performance by keeping frequently used data in RAM.

---

## 23. Common Mistakes

- **Setting no TTL** — the cache fills with data nobody reads and eventually runs out of memory.
- **Treating Redis as the source of truth** — writing data only to Redis, then being surprised when a restart loses it.
- **Caching everything** — data that is read once is pure overhead inside a cache.
- **Forgetting the miss path** — code that assumes the value is always there and crashes on `nil`.
- **Never invalidating** — the database changes, Redis keeps serving the old answer, and users see stale data.

---

## 24. Open Source Technologies

- **Redis** — the original
- **Valkey** — community fork created after the Redis licence change
- **Memcached** — minimal caching alternative
- **KeyDB**, **Dragonfly** — multi-threaded, Redis-protocol-compatible servers

---

## 25. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **Ideas for my own project:**
- **Useful links:**
- **To research later:**

---

## 26. Workbook Exercise

Pick one and actually do it:

- [ ] Draw the cache-hit and cache-miss paths by hand, without looking above.
- [ ] Explain in two sentences why Redis is faster than PostgreSQL.
- [ ] Name three things in your own project that are safe to cache, and one that is not — and say why.
- [ ] Compare Redis and Memcached: when would the simpler tool be the better choice?

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
User
    ↓
Browser
    ↓
Frontend
    ↓
FastAPI
    ↓
Redis
    ↓
PostgreSQL
```

## 2. Request Flow

```text
Input       key lookup  (GET product:42)
    ↓
Processing  in-memory hash table lookup, TTL check
    ↓
Output      cached value, or nil → fall back to the database
```

## 3. Real-World Usage

**Twitter / X** caches user timelines in Redis. Rebuilding a timeline from the database on every refresh would be far too slow at their read volume, and the very same timeline is requested repeatedly within seconds.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | An in-memory key-value store |
| **Why does it exist?** | Because database reads are slow and often repeated |
| **Where does it belong?** | Between the application layer and the database |
| **When should I use it?** | When the same data is read many times and slightly stale data is acceptable |
