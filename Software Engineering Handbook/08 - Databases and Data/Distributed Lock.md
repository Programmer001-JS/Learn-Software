# Distributed Lock

> **In one line —** making sure only one process across many machines does a thing at a time — and it is much harder to get right than it looks.

| | |
|---|---|
| **Category** | Coordination Pattern |
| **Architectural Layer** | Distributed coordination |
| **Related notes** | [Redis](Redis.md) · [Transactions](Transactions.md) · [Background Workers](../10%20-%20Distributed%20Systems/Background%20Workers.md) · [Multithreading](../02%20-%20Computer%20Science%20Fundamentals/Multithreading.md) |

---

## 1. Short Definition

*What is it?*

A distributed lock is a mechanism ensuring that, among many independent processes on many machines, only one holds the right to perform a particular operation at any moment.

---

## 2. Purpose

*What is its main purpose?*

To prevent duplicate or conflicting work when the same code runs simultaneously on several servers — which, once you scale horizontally, it always does.

---

## 3. Problem

*What engineering problem does it solve?*

```text
A cron job runs on 3 application servers
    ↓
All three fire at 02:00
    ↓
All three send the monthly invoice email
    ↓
Every customer receives three invoices
```

An in-process [mutex](../02%20-%20Computer%20Science%20Fundamentals/Multithreading.md) does nothing here — each server has its own memory. Coordination must happen somewhere both can see.

---

## 4. Architecture Position

```text
Worker 1     Worker 2     Worker 3
    └────────────┼────────────┘
                 ↓
        SET lock:invoices <token> NX EX 60
                 ↓
        ┌────────┴────────┐
    acquired            refused
        ↓                  ↓
    do the work        skip, or wait
        ↓
    release (only if the token still matches)
```

---

## 5. The naive implementation, and why it is wrong

```text
✗ NAIVE
    if not redis.get("lock"):        ← two processes can both read "no lock"
        redis.set("lock", 1)         ← both then set it
        do_work()                    ← both proceed
```

```text
✓ ATOMIC
    SET lock:job <random-token> NX EX 30
        NX  → only set if it does not exist   (atomic check-and-set)
        EX  → auto-expire, so a crashed holder does not block forever
```

> [!IMPORTANT]
> Two properties are non-negotiable: acquisition must be **atomic**, and the lock must **expire on its own**. A lock without a TTL, held by a process that crashes, blocks the operation permanently and requires manual intervention at 3am.

---

## 6. Releasing safely

```text
✗ DEL lock:job
     ↓
   You may be deleting SOMEONE ELSE'S lock:

   Worker A acquires, TTL 30s
   Worker A stalls for 35s (GC pause, slow query, network)
   Lock EXPIRES
   Worker B acquires it
   Worker A wakes up and calls DEL  →  releases B's lock
   Worker C acquires while B is still working  →  two workers, no lock
```

```text
✓ Release only if the token is still yours — atomically, via a Lua script:
     if redis.call("GET", key) == token then redis.call("DEL", key) end
```

---

## 7. The uncomfortable truth about correctness

> [!CAUTION]
> **A distributed lock cannot give you a hard mutual-exclusion guarantee.** The scenario above — the holder stalls past its TTL and does not know it — cannot be fully eliminated. Process pauses, network partitions and clock differences make it fundamentally unsolvable with timeouts alone.
>
> This is a well-known debate (Redlock and its critics), and the practical conclusion is clear:

```text
Lock for EFFICIENCY     "usually only one worker does this"
                        duplicate work is wasteful but harmless
                        → a Redis lock is fine

Lock for CORRECTNESS    "it is a bug if this happens twice"
                        money, inventory, uniqueness
                        → DO NOT rely on a distributed lock
                        → use a database transaction, a unique constraint,
                          or an idempotency key
```

---

## 8. Prefer the alternatives when correctness matters

| Instead of a lock | Use |
|---|---|
| Preventing duplicate charges | An **idempotency key** with a unique constraint |
| Preventing double booking | A **database transaction** with `SELECT ... FOR UPDATE` |
| Preventing duplicate rows | A **unique index** — the database enforces it absolutely |
| Distributing jobs | A **queue** — the broker guarantees one consumer per message |
| Running a job once | A **scheduler** designed for it, or a database row as the lock |

> [!TIP]
> `SELECT ... FOR UPDATE SKIP LOCKED` in PostgreSQL gives you correct, transactional work distribution with no extra system at all — and no lock-expiry problem, because the transaction owns the lock.

---

## 9. Real World Example

- **Scheduled jobs on multiple instances** — the most common legitimate use.
- **Preventing cache stampedes** — one process regenerates an expensive value while others wait.
- **Leader election** — one instance performs a coordinating role; Kubernetes uses this pattern.
- **Rate limiting a shared external API** across all workers.

---

## 10. Communication and Dependencies

- **A shared store** all participants can reach — Redis, ZooKeeper, etcd, or the database
- **Atomic primitives** — `SET NX EX`, or a compare-and-swap
- **A TTL** on every lock
- **A unique token** per acquisition

---

## 11. Alternatives

```text
Database row lock    SELECT ... FOR UPDATE — transactional, correct, single-DB
    ↓
Unique constraint    the database refuses the duplicate outright
    ↓
Redis SET NX EX      simple, fast, efficiency-grade only
    ↓
etcd / ZooKeeper     consensus-based, designed for coordination, heavier
    ↓
Queue                often removes the need for a lock entirely
```

---

## 12. When To Use / When NOT To Use

> [!TIP]
> Use a distributed lock when duplicate execution is **wasteful but not harmful**, and when there is no simpler database-level constraint available.

> [!CAUTION]
> Do not use one where a unique constraint, a transaction or a queue would do. Every distributed lock is a piece of coordination that can fail in ways that are hard to test and harder to reproduce.

---

## 13. Advantages and Disadvantages

**Advantages**
- Prevents most duplicate work with little code
- Fast, when built on Redis
- Works across languages and services
- Enables simple leader election

**Disadvantages**
- **No hard correctness guarantee**
- TTL tuning is a trade between safety and stuck locks
- Failure modes are subtle and rarely surface in testing
- Adds a dependency on the lock store
- A held lock serialises work you may have wanted parallel

---

## 14. Performance Impact

| Aspect | Impact |
|---|---|
| **Acquisition** | One Redis round trip — well under a millisecond |
| **Contention** | Waiting workers idle; use backoff, not tight polling |
| **TTL too short** | Lock expires mid-work — the dangerous case |
| **TTL too long** | A crashed holder blocks everyone for that duration |

> [!TIP]
> If work may exceed the TTL, **renew the lock periodically** from the holder (a "watchdog"). Redisson and similar libraries do this for you.

---

## 15. Security Considerations

> [!CAUTION]
> A lock store that any service can write to is a **denial-of-service vector**: acquiring a critical lock and never releasing it halts the operation it guards.

- **Authenticate access** to the lock store; it is infrastructure, not public
- **Use unpredictable tokens** so a lock cannot be released by guessing
- **Always set a TTL** — an indefinitely held lock is an outage waiting to happen
- **Monitor lock wait times**; a sudden rise usually means a stuck holder

---

## 16. Mental Model

> [!NOTE]
> **A distributed lock is a single key to a shared meeting room, hung on a hook in reception.**
>
> Whoever takes it may use the room. If they leave with the key in their pocket, nobody else can ever get in — so reception cuts a new key after an hour. Which means the original holder may still be inside when someone else walks in, believing the room to be free.
>
> That last sentence is the entire correctness problem, and it is why you do not put anything irreversible in that room.

---

## 17. Mini Architecture Diagram

```text
Worker A     Worker B     Worker C
    └────────────┼────────────┘
                 ↓
        Redis:  SET lock NX EX
                 ↓
        one acquires · others back off
                 ↓
        work (renew TTL if long-running)
                 ↓
        release, only if the token matches
```

---

## 18. Complete Request Flow

A scheduled job across three instances:

```text
02:00 — all three workers wake
    ↓
Each: SET lock:daily-report <uuid> NX EX 300
    ↓
Worker A succeeds; B and C receive nil and exit
    ↓
Worker A begins; a watchdog renews the TTL every 60 s
    ↓
Work completes
    ↓
Release via Lua: delete only if the stored token is still A's
    ↓
─────────── failure case ───────────
Worker A crashes mid-job
    ↓
TTL expires after 300 s
    ↓
Next scheduled run acquires the lock and proceeds
    ↓
The job must therefore be IDEMPOTENT — a partial run may be repeated
```

---

## 19. Key Takeaway

> [!IMPORTANT]
> Use distributed locks to avoid wasteful duplicate work, never to guarantee correctness — for that, use a database constraint, a transaction, or an idempotency key.

---

## 20. Common Mistakes

- **No TTL** — a crashed holder blocks the operation forever
- **`DEL` without checking the token** — releasing someone else's lock
- **Non-atomic acquisition** — check-then-set has a race by construction
- **Relying on a lock for financial correctness**
- **Assuming the job runs exactly once** — design it to be idempotent regardless
- **Tight polling loops** while waiting, hammering the lock store
- **Using a lock where a unique constraint or a queue would be simpler and stronger**

---

## 21. Open Source Technologies

- **Redis** with `SET NX EX`; **Redisson** and **redlock** libraries with watchdog renewal
- **etcd**, **ZooKeeper**, **Consul** — consensus-based coordination
- **PostgreSQL advisory locks** and `SELECT ... FOR UPDATE SKIP LOCKED`
- **Kubernetes leases** — leader election as a platform primitive

---

## 22. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 23. Workbook Exercise

- [ ] Find every scheduled job in your system and check whether it is safe to run twice.
- [ ] Identify one place using a lock where a unique constraint would be stronger.
- [ ] Verify that your lock release checks a token rather than deleting unconditionally.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Many workers → shared lock store (Redis/etcd) → one holder → guarded work
```

## 2. Request Flow

```text
Input       many processes attempting the same operation
    ↓
Processing  atomic acquire with a TTL; one wins, others back off
    ↓
Output      usually-single execution — never a hard guarantee
```

## 3. Real-World Usage

**Kubernetes leader election** uses lease objects for exactly this pattern: many controller replicas run, one acts. It is also a good illustration of the honest framing — the system is designed so that a brief overlap of two leaders is survivable.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Coordination so only one process across machines performs an operation |
| **Why does it exist?** | Because in-process locks are invisible to other servers |
| **Where does it belong?** | Around work that must not usefully be duplicated |
| **When should I use it?** | For efficiency — never as the correctness guarantee for money or uniqueness |
