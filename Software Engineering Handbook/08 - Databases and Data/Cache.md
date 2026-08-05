# Cache

> [!NOTE]
> **This note is one half of a pair.**
> **This note** covers caching as a *concept*: what a cache is, the read and write strategies, TTL and the invalidation problem.
> **[10 - Caching](../10%20-%20Distributed%20Systems/Caching.md)** covers caching as a *distributed system concern*: cache layers from browser to CDN to database, stampedes, consistency across many instances, and what breaks at scale.
> Read this one first.

> **In one line —** keeping a copy of an expensive answer somewhere cheap, and accepting that the copy will sometimes be wrong.

| | |
|---|---|
| **Category** | Architectural Pattern |
| **Architectural Layer** | Between the application and its data |
| **Related notes** | [Redis](Redis.md) · [Caching](../10%20-%20Distributed%20Systems/Caching.md) · [Cache Memory](../02%20-%20Computer%20Science%20Fundamentals/Cache%20Memory.md) · [Database Optimization](Database%20Optimization.md) |

---

## 1. Short Definition

*What is it?*

A cache stores the result of an expensive operation so that subsequent identical requests can be answered from the cheap copy instead of repeating the work.

---

## 2. Purpose

*What is its main purpose?*

To trade **memory and correctness** for **speed**. That trade is the whole subject: a cache is never free, and what it costs is the guarantee that your data is current.

---

## 3. Problem

*What engineering problem does it solve?*

```text
10,000 users open the same product page
    ↓
The same query runs 10,000 times
    ↓
The same answer is computed 10,000 times
    ↓
The database becomes the bottleneck for the whole system
```

---

## 4. Caching appears at every layer

The same idea, at wildly different scales — this is one of computing's most repeated patterns.

```text
CPU cache            64 bytes      ~1 ns
Application memory   MBs           ~0.1 µs
Redis                GBs           ~1 ms
CDN                  TBs           ~10 ms
    ↓
Each layer caches the layer below it
```

---

## 5. Read strategies

```text
CACHE-ASIDE  (lazy loading)   ← the default; you control it
    read: check cache → miss → read DB → write to cache → return
    ✓ only caches what is actually requested
    ✗ every first request pays full cost

READ-THROUGH
    the cache itself fetches from the database on a miss
    ✓ simpler application code
    ✗ requires cache support

REFRESH-AHEAD
    refresh popular entries before they expire
    ✓ users never hit a cold miss
    ✗ refreshes data nobody asked for
```

---

## 6. Write strategies

```text
WRITE-THROUGH        write to cache AND database, synchronously
                     ✓ cache always consistent   ✗ slower writes

WRITE-BEHIND         write to cache now, database later
                     ✓ very fast writes          ✗ DATA LOSS on crash

WRITE-AROUND         write to database, invalidate the cache entry
                     ✓ avoids caching write-once data  ✗ next read misses

CACHE INVALIDATION   write to database, delete the cache key
                     ✓ simple and safe           ← the usual correct choice
```

> [!TIP]
> **Delete, do not update.** On a write, removing the cache key is safer than writing the new value into it. Two concurrent updates can interleave and leave a stale value in the cache permanently; a deleted key simply causes the next read to fetch fresh data.

---

## 7. TTL — the pragmatic answer to invalidation

> [!IMPORTANT]
> Every cache entry should have a **time to live**. A TTL converts "this might be wrong forever" into "this might be wrong for at most 5 minutes" — which is a bounded, explainable problem rather than an unbounded one.

```text
Rapidly changing, tolerant of staleness   →  30–60 s
Product and content pages                 →  5–15 min
Reference data, rarely changing           →  hours
Never expires                             →  a bug waiting to happen
```

---

## 8. What to cache, and what not to

```text
✓ Read many times, changes rarely
✓ Expensive to compute
✓ Same for many users
✓ Slightly stale is acceptable

✗ Unique per request         — the cache is never hit
✗ Changes constantly         — invalidation costs more than the saving
✗ Must always be exact       — balances, stock levels, permissions
✗ Very large objects         — memory cost outweighs the benefit
```

---

## 9. The hit rate decides everything

```text
Hit rate 95%  →  the database sees 5% of the traffic     ← excellent
Hit rate 50%  →  half the traffic, plus cache overhead   ← questionable
Hit rate 10%  →  you added a system and gained nothing   ← remove it
```

> [!CAUTION]
> **Measure your hit rate.** A cache nobody hits is pure cost: extra latency, extra memory, extra failure mode, and a stale-data risk with no compensating benefit.

---

## 10. Real World Example

- **[Redis](Redis.md)** in front of PostgreSQL is the canonical application cache.
- **HTTP `Cache-Control` headers** cache in the browser and at the CDN — free, and frequently unused.
- **ORM query caches and identity maps** cache within a single request.
- **Memoisation** is caching inside one function call.

---

## 11. Communication and Dependencies

- **A cache store** — Redis, Memcached, or in-process memory
- **A source of truth behind it** — a cache without one is not a cache
- **An invalidation path** — something must delete keys when data changes

---

## 12. When To Use / When NOT To Use

> [!TIP]
> Cache when the same expensive answer is requested repeatedly and being a little out of date is acceptable. Start with a TTL and cache-aside; add sophistication only when measurements demand it.

> [!CAUTION]
> Do not cache to hide a slow query — fix the query first. A missing index served from a cache is still a missing index, and the first cold request, the cache restart, and the eventual scale-up will all reveal it.

---

## 13. Advantages and Disadvantages

**Advantages**
- Order-of-magnitude latency improvement on hits
- Large reduction in database load
- Absorbs traffic spikes
- Often the cheapest available performance win

**Disadvantages**
- **Stale data** — the unavoidable cost
- **Invasive invalidation logic** spread across the codebase
- Another system to run, monitor and secure
- Hard-to-reproduce bugs: "it works for me" because a cache differs
- Memory cost, and eviction behaviour to reason about

---

## 14. Eviction

When memory fills, something must go:

```text
LRU   least recently used      ← the sensible default
LFU   least frequently used    good for stable popularity
TTL   expire by age            simplest and most predictable
FIFO  oldest first             rarely what you want
```

---

## 15. Security Considerations

> [!CAUTION]
> **Caching a response that includes a user's private data, under a key that is not user-specific, serves one user's data to another.** This is a real and recurring incident class, especially with CDN and reverse-proxy caching of authenticated pages.

- **Include the user or tenant in the cache key** for anything personalised
- **Never cache authenticated responses at a shared layer** without `Cache-Control: private`
- **Cached data inherits the sensitivity of its source** — a cache of user profiles is personal data
- **Invalidate on permission change**, or a revoked user keeps seeing cached content they may no longer access
- **The cache is a separate system to secure** — see [Redis](Redis.md) security

---

## 16. Mental Model

> [!NOTE]
> **A cache is a photocopy on your desk.**
>
> Far faster than walking to the archive. The risk is exactly the obvious one: someone updated the original and you are reading last week's copy. A TTL is writing "discard after Friday" on it — you accept being wrong, but only for a bounded time.

---

## 17. Mini Architecture Diagram

```text
Request
    ↓
Cache lookup
    ├── HIT  → return immediately
    └── MISS → source of truth
                    ↓
             store with a TTL
                    ↓
                 return
```

---

## 18. Complete Request Flow

Read, then write:

```text
GET /product/42
    ↓
Redis GET product:42
    ↓ HIT  → return  (~1 ms, database untouched)
    ↓ MISS
SELECT from PostgreSQL  (~40 ms)
    ↓
SET product:42 ... EX 300
    ↓
Return

──────── later: the product is edited ────────

UPDATE products SET price = ... WHERE id = 42
    ↓
DEL product:42          ← delete, do not update
    ↓
Next read misses, fetches fresh, repopulates
```

---

## 19. Key Takeaway

> [!IMPORTANT]
> A cache trades correctness for speed — always set a TTL, delete rather than update on writes, and measure the hit rate to confirm the trade was worth making.

---

## 20. Common Mistakes

- **No TTL** — stale data forever, and memory that never frees
- **Updating cache entries on write** instead of deleting them
- **Caching per-user data under a shared key** — a data-leak bug
- **Caching to hide a slow query** rather than fixing it
- **Never measuring the hit rate**
- **No plan for a cold cache** — a restart sends 100% of traffic to the database at once
- **Caching everything**, including data read once

---

## 21. Open Source Technologies

- **Redis**, **Valkey**, **Memcached** — cache stores
- **HTTP caching** — `Cache-Control`, `ETag`, and every CDN
- **cachetools** (Python), **lru-cache** (Node), **Caffeine** (Java) — in-process caches
- **Varnish** — a dedicated HTTP cache

---

## 22. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 23. Workbook Exercise

- [ ] Measure the hit rate of one cache in your system. Is it above 80%?
- [ ] Find one cache key that could contain user-specific data without a user identifier in the key.
- [ ] Check what happens to your database load if the cache is flushed at peak traffic.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Client → Application → Cache → Database
```

## 2. Request Flow

```text
Input       a request for data
    ↓
Processing  cache lookup; on a miss, compute and store with a TTL
    ↓
Output      an answer, possibly slightly out of date by a bounded amount
```

## 3. Real-World Usage

**HTTP caching** is the most widely deployed and most underused cache in existence. Correct `Cache-Control` headers on static assets move them to the user's own machine — the cheapest possible latency improvement.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A fast copy of an expensive answer |
| **Why does it exist?** | Because recomputing identical results is wasteful |
| **Where does it belong?** | Between a consumer and an expensive source |
| **When should I use it?** | Repeated reads where bounded staleness is acceptable |
