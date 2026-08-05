# Caching — Distributed Systems

> [!NOTE]
> **This note is one half of a pair.**
> **[08 - Cache](../08%20-%20Databases%20and%20Data/Cache.md)** covers caching as a *concept*: read and write strategies, TTL, invalidation, hit rate. **Read that one first.**
> **This note** covers what changes once there are many machines: the layers from browser to CDN, cache stampedes, consistency between instances, and the failure modes that only appear at scale.

> **In one line —** at one server a cache is an optimisation; across many servers it becomes a distributed system with its own failure modes.

| | |
|---|---|
| **Category** | Distributed Systems Pattern |
| **Architectural Layer** | Spanning client, edge and server |
| **Related notes** | [Cache](../08%20-%20Databases%20and%20Data/Cache.md) · [Redis](../08%20-%20Databases%20and%20Data/Redis.md) · [Load Balancing](../14%20-%20Scalability%20and%20Reliability/Load%20Balancing.md) · [HTTP HTTPS](../04%20-%20Networking%20and%20Internet/HTTP%20HTTPS.md) |

---

## 1. The cache layers

A request may be answered at any of these, and the earlier it stops, the cheaper it is.

```text
Browser cache          0 ms      per user      ← free, most underused
    ↓
CDN / edge             ~10 ms    per region
    ↓
Reverse proxy (Nginx)  ~1 ms     per data centre
    ↓
Application memory     ~0.1 µs   PER INSTANCE  ← the source of inconsistency
    ↓
Redis                  ~1 ms     shared
    ↓
Database buffer pool   ~0.1 µs   inside the database
    ↓
Disk
```

> [!IMPORTANT]
> **Each layer has a different scope, and that is what makes multi-layer caching hard.** Invalidating Redis does nothing to the copy sitting in a CDN in Frankfurt, or in a user's browser, or in the local memory of eleven application instances.

---

## 2. The local-memory trap

```text
ONE SERVER                      TEN SERVERS
in-process cache                ten in-process caches
    ↓                               ↓
always consistent               ten different answers to the same question
    ↓                               ↓
fine                            a user refreshes and sees the price change
                                back and forth depending on routing
```

> [!CAUTION]
> In-process caching is extremely fast and silently breaks correctness the moment you scale out. Either accept a bounded staleness deliberately (short TTL, tolerant data), or use a **shared** cache such as [Redis](../08%20-%20Databases%20and%20Data/Redis.md).

---

## 3. Cache stampede — the failure that takes systems down

```text
A popular key expires
    ↓
10,000 concurrent requests all miss simultaneously
    ↓
ALL of them query the database
    ↓
The database saturates and slows
    ↓
Requests time out, retry, and miss again
    ↓
Outage
```

This is also called the **thundering herd**, and it is the most common way a cache causes an incident rather than preventing one.

**Three defences:**

```text
LOCKING          the first miss acquires a lock and regenerates;
                 the others wait briefly and read the new value

STALE-WHILE-     serve the expired value while ONE request refreshes
REVALIDATE       in the background — no user waits at all

JITTERED TTL     expire at 300 s ± random(60) so keys do not
                 all expire at the same instant
```

> [!TIP]
> **Jitter is one line of code and prevents the synchronised-expiry version of this problem entirely.** If you take one thing from this note, take that.

---

## 4. Cold cache after a restart

```text
Cache flushed or restarted
    ↓
Hit rate drops from 95% to 0%
    ↓
The database receives 20× its normal load, instantly
    ↓
Often worse than having no cache at all
```

> [!CAUTION]
> A system that only survives because of its cache has a hidden dependency. Ask: **can the database survive a cold cache at peak traffic?** If not, the cache is load-bearing infrastructure and must be treated as such — replicated, monitored and warmed after deployment.

---

## 5. CDN caching

The highest-leverage cache layer, and the one most often left unconfigured.

```text
Cache-Control: public, max-age=31536000, immutable
    ↓  for hashed static assets (app.a3f9c2.js)
Never revalidated. Changing the file changes its name.

Cache-Control: public, max-age=300, stale-while-revalidate=3600
    ↓  for semi-dynamic pages
Served instantly; refreshed in the background.

Cache-Control: private, no-store
    ↓  for anything user-specific        ← see the security section
```

**Purging is the hard part.** A CDN holds copies in hundreds of locations; invalidation is eventually consistent and can take minutes. This is why **content-hashed filenames** are the standard approach — you never invalidate, you change the URL.

---

## 6. Consistency models

```text
WRITE-THROUGH        cache and database updated together
                     consistent, slower writes, and racy across instances

INVALIDATE-ON-WRITE  delete the key on write        ← the usual correct choice
                     next read repopulates from the source

TTL ONLY             accept staleness up to the TTL
                     simplest, and honest about what it guarantees

PUB/SUB INVALIDATION one instance writes and publishes "invalidate X";
                     all instances drop their local copy
```

> [!TIP]
> [Pub/Sub invalidation](../08%20-%20Databases%20and%20Data/Pub%20Sub.md) is how you make local in-memory caches survivable across instances — with the caveat that pub/sub delivery is not guaranteed, so a TTL must still bound the damage.

---

## 7. Cache key design

```text
✗ "user_data"                        collides across users — a data leak
✓ "user:42:profile:v3"

Include:  entity · id · variant · SCHEMA VERSION
```

> [!TIP]
> **A version component in the key is the cheapest invalidation mechanism that exists.** Change the shape of what you cache, bump `v3` to `v4`, and every old entry is orphaned instantly — with no purge, no scan and no race.

---

## 8. Real World Example

- **Content-hashed asset filenames** (`app.a3f9c2.js`) let every CDN cache them forever, which is why modern build tools produce them by default.
- **Stale-while-revalidate** is what makes news and e-commerce sites feel instant under load.
- **Facebook's leases** and similar mechanisms exist specifically to solve the stampede problem at scale.

---

## 9. When To Use / When NOT To Use

> [!TIP]
> Push caching as far toward the user as the data allows. A browser cache hit costs nothing and travels no distance; a Redis hit still crosses a network.

> [!CAUTION]
> Do not cache at multiple layers without knowing how each is invalidated. A bug where "the change is live but some users see the old version for an hour" is almost always a layer someone forgot about.

---

## 10. Advantages and Disadvantages

**Advantages**
- Absorbs traffic spikes across every layer
- Enormous reduction in origin load
- Geographic distribution reduces latency in a way nothing else can
- Often the cheapest scaling available

**Disadvantages**
- Multi-layer invalidation is genuinely hard
- Stampedes and cold-start effects can cause outages
- Debugging becomes harder — behaviour depends on which layer answered
- Shared caches at the edge are a real data-leak risk

---

## 11. Performance Impact

| Layer | Latency | Invalidation difficulty |
|---|---|---|
| **Browser** | 0 ms | Impossible — only expiry |
| **CDN** | ~10 ms | Minutes, eventually consistent |
| **Reverse proxy** | ~1 ms | Easy, per data centre |
| **In-process** | ~0.1 µs | Hard across instances |
| **Redis** | ~1 ms | Easy and immediate |

---

## 12. Security Considerations

> [!CAUTION]
> **Web cache poisoning and cache deception are real attack classes.** If a shared cache stores a response containing one user's data under a key that does not include the user, the next visitor receives it. This has produced genuine incidents at major sites.

- **`Cache-Control: private, no-store`** on anything authenticated — and verify the CDN honours it
- **Vary correctly** — an incorrectly configured `Vary` header lets one user's response be served to another
- **Never let unkeyed request headers influence a cached response** — that is the poisoning vector
- **Cache deception**: `/account/statement.css` may be treated as a cacheable static file by the CDN while your application still returns account data. Normalise paths and be explicit about what is cacheable
- **Purge cached data on permission change**, or a revoked user keeps seeing content from the cache

---

## 13. Mental Model

> [!NOTE]
> **Multi-layer caching is photocopies distributed across a company.**
>
> Head office updates the original. The regional offices, each department and every individual desk still hold their own copies, made at different times. Recalling them all is far harder than updating the original — which is why "print an expiry date on every copy" (a TTL) is the strategy that actually works.

---

## 14. Mini Architecture Diagram

```text
User
 ↓ browser cache        (per user, cannot be purged)
CDN edge                (per region, minutes to purge)
 ↓
Reverse proxy           (per data centre)
 ↓
App instance ×N  — local memory caches, INCONSISTENT with each other
 ↓                        ↑ pub/sub invalidation
Redis                   (shared, authoritative cache)
 ↓
Database
```

---

## 15. Complete Request Flow

```text
Request for a product page
    ↓
Browser cache fresh?     → 0 ms, no network at all
    ↓ no
CDN edge hit?            → ~10 ms
    ↓ miss
Reverse proxy hit?       → ~1 ms
    ↓ miss
App instance: local memory?  → instant, but possibly stale
    ↓ miss
Redis GET                → ~1 ms
    ↓ MISS — and 10,000 requests arrive at once
STAMPEDE PROTECTION: one request acquires a lock and regenerates;
                     the rest serve the stale value
    ↓
Database queried ONCE
    ↓
Redis populated with a JITTERED TTL
    ↓
Response cached back down the chain with correct Cache-Control
```

---

## 16. Key Takeaway

> [!IMPORTANT]
> Across many machines, caching stops being an optimisation and becomes a distributed system — jitter your TTLs, protect against stampedes, and know how every layer is invalidated.

---

## 17. Common Mistakes

- **In-process caches across multiple instances**, producing inconsistent answers
- **No jitter**, so a whole key class expires simultaneously
- **No stampede protection** on expensive keys
- **A system that cannot survive a cold cache**
- **Caching authenticated responses** at a shared layer
- **Forgetting a layer** when invalidating
- **No version component** in cache keys
- **Treating CDN purge as instant**

---

## 18. Open Source Technologies

- **Redis**, **Valkey**, **Memcached** — shared caches
- **Varnish**, **Nginx proxy_cache** — reverse proxy caching
- **Cloudflare**, **Fastly** — CDN, with `stale-while-revalidate` support
- **cachetools**, **Caffeine** — in-process caches with TTL and size limits
- **`Cache-Control`, `ETag`, `Vary`** — the standard everyone already has and rarely uses fully

---

## 19. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 20. Workbook Exercise

- [ ] List every cache layer in your system and, for each, write down how it is invalidated.
- [ ] Check whether your TTLs have jitter.
- [ ] Estimate your database load if the cache were flushed at peak traffic.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Browser → CDN → proxy → app instances → Redis → database
```

## 2. Request Flow

```text
Input       a request that may be answered at any layer
    ↓
Processing  checked outward-in; on a miss, regenerated once and repopulated
    ↓
Output      a response, plus copies at several layers with different lifetimes
```

## 3. Real-World Usage

**Content-hashed filenames** solved CDN invalidation by removing it: assets are cached forever because a change produces a new URL. Sidestepping invalidation entirely, rather than solving it, is the most successful pattern in this area.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Caching across multiple layers and multiple machines |
| **Why does it exist?** | Because origin capacity and the speed of light are both limits |
| **Where does it belong?** | Everywhere between the user and the database |
| **When should I use it?** | When you can answer, for every layer, how it gets invalidated |
