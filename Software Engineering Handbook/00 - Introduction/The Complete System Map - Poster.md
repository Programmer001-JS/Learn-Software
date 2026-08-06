# The Complete System Map — Poster

> **In one line —** the same journey as [The Complete System Map](The%20Complete%20System%20Map.md), drawn wide as a single route across the page; this file is for the wall, that one is for looking things up.

| | |
|---|---|
| **Category** | Map *(wide format — built for printing)* |
| **Width** | Every diagram is **≤ 134 columns**. A3 landscape at 10 pt, or A4 landscape at 7–8 pt. |
| **Companion** | [The Complete System Map](The%20Complete%20System%20Map.md) — the station-by-station detail and all the links |
| **Related notes** | [README](README.md) · [How To Use This Handbook](How%20To%20Use%20This%20Handbook.md) · [Diagrams](../17%20-%20Personal%20Knowledge%20Base/Diagrams.md) |

---

## How to print this

```text
EVERY DIAGRAM HERE IS EXACTLY ≤ 134 CHARACTERS WIDE.
That number is the whole design constraint — it is what makes paper possible.

    A3 LANDSCAPE   monospace 10 pt   → comfortable, this is the intended size
    A4 LANDSCAPE   monospace 7-8 pt  → fits, readable, tight
    A4 PORTRAIT    → do not. It will wrap and the map becomes noise.

THE RELIABLE WAY (any operating system, no tooling):
    1. open this file in a plain text editor
    2. TURN WORD WRAP OFF          ← if you skip this, nothing else matters
    3. select one poster block, copy it into a blank document
    4. set the font to a monospace one (Consolas, Courier New, JetBrains Mono)
    5. page setup → LANDSCAPE → shrink the font until no line breaks
    6. print

ONE POSTER PER SHEET. Do not try to fit two.
```

> [!CAUTION]
> **Word wrap is the only thing that can ruin this.** ASCII diagrams carry their meaning in column alignment; a single wrapped line shifts everything below it and the map stops being readable. If a printed line breaks, the font is too large — shrink it, do not narrow the margins.

> [!TIP]
> **Print POSTER 1 and POSTER 5 first, and put them side by side.** One is the route, the other is what waits on the route. Together they are the two things worth having on a wall; the other three are worth having in a drawer.

---

# POSTER 1 — THE JOURNEY

**One request, one route, left to right and back again.**

```text
                            T H E   J O U R N E Y   O F   O N E   R E Q U E S T
      from a click on a device you do not own, to a product in a hand you cannot see — and back, in under a second

╔════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════╗
║ IDENTITY   ·   who is allowed to do this?   ·   IAM · authN · authZ · secrets                                                      ║
║                ▸ there is no perimeter to be inside of — IN THE CLOUD, IDENTITY IS THE PERIMETER                                   ║
╚════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════╝

        DNS           HTTP req      CDN / edge    load bal.     runtime       routing       authZ         LOGIC         ORM
        name → IP     built         ★ EXIT 1      healthy?      proc/thread   controller    may you?      your product  → SQL
        ▼             ▼             ▼             ▼             ▼             ▼             ▼             ▼             ▼
 01     02     03     04     05     06     07     08     09     10     11     12     13     14     15     16     17     18     19
 ●──────●──────●──────●──────●──────●──────●──────●──────●──────●──────●──────●──────●──────●──────●──────●──────●──────●──────●
 ▲             ▲             ▲             ▲             ▲             ▲             ▲             ▲             ▲             ▲
 the click     TCP + TLS     ISP → net     VPC edge      proxy         middleware    authN         validate      cache        DATABASE
 JS runs       handshake     physics       route table   nginx         the chain     who are you?  UNTRUSTED!    ★ EXIT 2    the truth
 └─ THEIR DEVICE ───────────┘└─ NET ──────┘└─ YOUR CLOUD ──────┘└─ YOUR APPLICATION ────────────────────────────┘└─ YOUR DATA ───────┘
                                    │                                                                            │
                                    └────────────────────────────────────────────────────────────────────────────┘
                                                                          ▼
┌────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ THE TWO SHORTCUTS   ★ EXIT 1 at the edge (06)      ★ EXIT 2 in the cache (17)                                                      │
│ A well-built system answers MOST requests at one of them and NEVER reaches the database.                                           │
│ That is all performance work is. Everything to the right of EXIT 2 costs you real money.                                           │
└────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ THE FLOOR  —  not a station. This is the ground under ALL of them.                                                                 │
│ CPU · RAM · SSD · kernel · syscalls · scheduler · context switching · memory management · file systems                             │
│ ▸ this layer is WHY the advice above is true: caching works because RAM ≫ disk, event loops stall                                  │
│   because of the scheduler, a container is light because it shares a kernel                                                        │
└────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘

                                                             the response does NOT wait for any of this  →│
                                                                                                          ▼
                                                ┌────────────────────────────────────────────────────────────────────────────────────┐
                                                │ 21   TOO SLOW FOR A REQUEST  —  it goes on a queue and the user is answered NOW    │
                                                │      queue → worker → email · SMS · media · AI · invoices · reports · webhooks     │
                                                │      ⚠ the moment work is RETRIED, everything it touches must be IDEMPOTENT        │
                                                │      → the whole continent is POSTER 2                                             │
                                                └────────────────────────────────────────────────────────────────────────────────────┘

 ◄──────●─────────────────────────────────────────●────────────────────────────────────────────────●
        24                                        23                                               22   the response turns around here
        ▲                                         ▲                                                ▲
        browser                                   back out + EGRESS                                serialise
        parse · render · paint                    proxy → LB → edge                                JSON / HTML
        ← THE USER'S CLOCK IS                     METERED PER GB OUT                               status code
          STILL RUNNING HERE                      ← the surprise bill

    a 45 ms API inside a 4-second page is a 4-second product — station 24 is where the user actually lives

╔════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════╗
║ OBSERVABILITY  ·  ONE request id, created at 11 and carried to 24  ·  metrics = alarm, logs = investigate,                         ║
║                   traces = find the slow hop  ·  alert on SYMPTOMS (users affected), never on CPU                                  ║
╠════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════╣
║ RELIABILITY    ·  a TIMEOUT and a FALLBACK on every arrow above  ·  two availability zones, each able to                           ║
║                   serve peak ALONE  ·  a rollback someone has actually used  ·  replication ≠ backup                               ║
╠════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════╣
║ COST           ·  compute · EGRESS at 23 · per-request charges · LOG INGESTION · idle capacity                                     ║
║                   autoscaling with no upper bound turns a traffic attack into a financial one                                      ║
╚════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════╝
```

**Station detail and every link:** [The Complete System Map](The%20Complete%20System%20Map.md)

---

# POSTER 2 — THE ASYNC CONTINENT

**Everything that happens after the user already has their answer.**

```text
                        W H A T   H A P P E N S   A F T E R   T H E   U S E R   L E A V E S


   station 21 of POSTER 1                                                                    THE TWO METRICS THAT MATTER
   dropped a message and returned                                                            ┌───────────────────────────┐
            │                                                                                │  QUEUE DEPTH              │
            ▼                                                                                │  OLDEST MESSAGE AGE       │
   ╔═══════════════════════╗                                                                 │  ← alarm on these, not CPU│
   ║        QUEUE          ║ ◄───────────────────────────────────────────────────────────────  └───────────────────────────┘
   ║  SQS · Kafka · Rabbit ║
   ║  Redis Streams        ║          workers scale on DEPTH, and scale to ZERO when idle
   ╚═══════════╦═══════════╝                     ↑ this is where the cost saving lives
               ║
       ┌───────╬────────┬────────────────┬────────────────┬────────────────┐
       ▼       ▼        ▼                ▼                ▼                ▼
   ┌────────┐┌────────┐┌────────┐   ┌────────┐       ┌────────┐       ┌────────┐
   │ worker ││ worker ││ worker │   │ worker │  ...  │ worker │       │ worker │      ON SPOT / PREEMPTIBLE CAPACITY
   │  spot  ││  spot  ││  spot  │   │  spot  │       │  spot  │       │  spot  │      up to ~90% cheaper, because the
   └────┬───┘└────┬───┘└────┬───┘   └────┬───┘       └────┬───┘       └────┬───┘      work is interruptible and retried
        │         │         │            │                │                │
        │    ⚠ EVERY WORKER MUST BE IDEMPOTENT — the same message WILL arrive twice ⚠
        │         │         │            │                │                │
        ▼         ▼         ▼            ▼                ▼                ▼
 ┌──────────────┐ ┌──────────────────┐ ┌────────────────────┐ ┌──────────────────────┐ ┌────────────────────────────────┐
 │ NOTIFY       │ │ MEDIA            │ │ AI / ML            │ │ DOCUMENTS            │ │ INTEGRATION                    │
 │ email · SMS  │ │ transcode        │ │ embed → retrieve   │ │ invoices · exports   │ │ outbound webhooks              │
 │ push         │ │ thumbnails       │ │ rerank → generate  │ │ reports · PDFs       │ │ third-party sync               │
 │              │ │ segments → S3    │ │ vector store       │ │                      │ │ retries with backoff           │
 │              │ │ → CDN            │ │ cost cap + iter cap│ │                      │ │                                │
 └──────┬───────┘ └────────┬─────────┘ └─────────┬──────────┘ └──────────┬───────────┘ └───────────────┬────────────────┘
        │                  │                     │                       │                             │
        └──────────────────┴──────────┬──────────┴───────────────────────┴─────────────────────────────┘
                                      ▼
                        ┌───────────────────────────────┐              ┌──────────────────────────────────────────┐
                        │  write the result to the DB   │              │  failed 3 times?                         │
                        │  notify the user — who may    │  ────────►   │  ╔════════════════════════════════════╗  │
                        │  well have closed the tab     │              │  ║  DEAD LETTER QUEUE → ALARM → HUMAN ║  │
                        └───────────────────────────────┘              │  ╚════════════════════════════════════╝  │
                                                                       │  without this, failures VANISH silently   │
                                                                       └──────────────────────────────────────────┘

   ┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
   │  EVERY BOX ON THIS MAP IS THE SAME SHAPE:  accept fast → queue → process idempotently → store → notify                     │
   │  Change the words and it is video, documents, AI or invoices. Learn the shape once; all of them are the same system.        │
   └─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

# POSTER 3 — THE DELIVERY ROAD

**None of POSTER 1 exists until something puts it there.**

```text
                    H O W   T H E   C O D E   G O T   T H E R E   —   a n d   h o w   i t   c o m e s   b a c k


  editor          GIT                PULL REQUEST            ARTEFACT            INFRASTRUCTURE      DEPLOY            PRODUCTION
    │              │                      │                      │                     │               │                   │
    ●──────────────●──────────────────────●──────────────────────●─────────────────────●───────────────●───────────────────●
    │              │                      │                      │                     │               │                   │
    ▼              ▼                      ▼                      ▼                     ▼               ▼                   ▼
 ┌──────┐  ┌───────────────┐  ┌────────────────────────┐  ┌───────────────┐  ┌────────────────┐  ┌──────────────┐  ┌──────────────┐
 │ you  │  │ immutable     │  │ lint · types           │  │ BUILT ONCE    │  │ Terraform      │  │ rolling      │  │ this is now  │
 │ write│  │ snapshots     │  │ unit tests             │  │               │  │ plan on the PR │  │ canary 5%    │  │ stations     │
 │ code │  │ branch = a    │  │ security scan   ┐      │  │ image:<sha>   │  │ apply on merge │  │ blue/green   │  │ 08 – 16 of   │
 │      │  │ movable label │  │ build           ├ PARA │  │               │  │                │  │              │  │ POSTER 1     │
 │      │  │               │  │ integration     ┘ LLEL │  │ SAME BYTES to │  │ builds the VPC,│  │ metrics OK?  │  │              │
 │      │  │ SHORT-LIVED   │  │                        │  │ staging AND   │  │ LB, database,  │  │  → promote   │  │              │
 │      │  │ branches, or  │  │ HUMAN REVIEW           │  │ production    │  │ cluster of     │  │  → or REVERT │  │              │
 │      │  │ merges hurt   │  │ ← small diffs only     │  │               │  │ POSTER 1       │  │              │  │              │
 │      │  │               │  │   get REAL review      │  │ signed        │  │                │  │ feature stays│  │              │
 │      │  │               │  │                        │  │               │  │ OIDC — NO      │  │ OFF behind   │  │              │
 │      │  │               │  │ main is PROTECTED,     │  │               │  │ stored cloud   │  │ a FLAG       │  │              │
 │      │  │               │  │ admins included        │  │               │  │ keys anywhere  │  │              │  │              │
 └──────┘  └───────────────┘  └────────────────────────┘  └───────────────┘  └────────────────┘  └──────────────┘  └──────────────┘


  ╔═══════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════╗
  ║  THE ROAD RUNS BOTH WAYS — and the return journey is the point                                                              ║
  ║                                                                                                                             ║
  ║      a feature flag      ── seconds        ← the fastest rollback there is, and it needs no pipeline                         ║
  ║      the previous image  ── minutes        ← because you built it once and never deleted it                                 ║
  ║      a database migration ── ⚠ DOES NOT COME BACK ⚠                                                                        ║
  ║                                                                                                                             ║
  ║      So: expand → migrate → contract, across THREE releases. Never ship a destructive migration with the code that needs it. ║
  ╚═══════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════╝
```

---

# POSTER 4 — THE FOUR BANDS

**Not stations. They touch every station, which is why they are drawn as bands.**

```text
                    W H A T   W R A P S   E V E R Y   S T A T I O N   O N   P O S T E R   1


            01   02   03   04   05   06   07   08   09   10   11   12   13   14   15   16   17   18   19   ...   22   23   24
             │    │    │    │    │    │    │    │    │    │    │    │    │    │    │    │    │    │    │           │    │    │
 ╔═══════════╪════╪════╪════╪════╪════╪════╪════╪════╪════╪════╪════╪════╪════╪════╪════╪════╪════╪════╪═══════════╪════╪════╪══╗
 ║ IDENTITY  ·  who is allowed to do this?                                                                                     ║
 ║           IAM · authentication · authorisation · secrets management · least privilege                                        ║
 ║           ▸ there is no perimeter to be inside of. A credential works from anywhere on earth.                                ║
 ║           ▸ the most common real flaw: authentication checked, AUTHORISATION FORGOTTEN                                       ║
 ╠══════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════╣
 ║ OBSERVABILITY  ·  what is happening, and is it acceptable?                                                                   ║
 ║           metrics → ALARM       logs → INVESTIGATE       traces → FIND THE SLOW HOP                                          ║
 ║           SLI → SLO → error budget → burn-rate alert → on-call + a RUNBOOK → blameless review                                ║
 ║           ▸ ONE request id, created at station 11, carried to station 24. Cheapest decision on the page.                     ║
 ║           ▸ alert on SYMPTOMS (users affected), never on CAUSES (CPU at 85%)                                                 ║
 ╠══════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════╣
 ║ RELIABILITY & SCALE  ·  under load, and when a part dies?                                                                    ║
 ║           vertical first (a bigger machine, no code change) → then horizontal (statelessness required)                        ║
 ║           two availability zones, each able to serve peak ALONE  ·  HA ≠ backups  ·  replication copies destruction          ║
 ║           ▸ a TIMEOUT and a FALLBACK on every single arrow of POSTER 1                                                       ║
 ║           ▸ above ~80% utilisation, latency stops being linear. Headroom is a feature.                                       ║
 ╠══════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════╣
 ║ COST  ·  what does each station cost, and which one surprises you?                                                           ║
 ║           compute · EGRESS (station 23) · per-request charges · LOG INGESTION · NAT gateways · idle capacity                 ║
 ║           ▸ egress and log volume are the two line items that catch out nearly every team                                    ║
 ║           ▸ autoscaling with no upper bound turns a traffic attack into a financial one                                      ║
 ║           ▸ an AI agent loop with no iteration cap is an unbounded bill, and it WILL find a way to loop                      ║
 ╚══════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════╝
```

---

# POSTER 5 — THE HAZARD MAP

**The same route, read as an adversary reads it. Print this one and keep it.**

```text
                        W H A T   W A I T S   F O R   Y O U   A T   E A C H   S T A T I O N


 ┌─────┬───────────────────┬──────────────────────────────────────────────────────────────────────────────────────────────────┐
 │  #  │ STATION           │ WHAT WAITS THERE                                                                                 │
 ├─────┼───────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────────┤
 │ 01  │ browser           │ a blocked event loop freezes the page before the request is even sent · no loading state         │
 │ 02  │ DNS               │ a TTL of hours, so failover takes hours · a client library that resolves once, forever           │
 │ 03  │ TLS               │ ⚠ AN EXPIRED CERTIFICATE — one of the most common self-inflicted outages there is                │
 │ 04  │ the request       │ a session token in localStorage, plus one XSS, equals account takeover                           │
 │ 05  │ the network        │ packet loss presents as inexplicable slowness · you cannot fix this layer, only design around it│
 │ 06  │ CDN / edge        │ nothing cached, so every request travels the whole route · a PERSONALISED page cached for all    │
 │ 07  │ VPC               │ a database reachable from the internet · 0.0.0.0/0 on an admin port · one availability zone      │
 │ 08  │ load balancer     │ ★ a DEEP health check removing EVERY backend when one dependency is slow → total outage         │
 │     │                   │   · idle timeout cutting a long request · draining shorter than the slowest request             │
 │     │                   │   · retries amplifying an overload into a collapse                                              │
 │ 09  │ proxy             │ a body size limit rejecting uploads · too few workers, so requests queue invisibly               │
 │ 10  │ runtime           │ one blocking call stalls everything · a thread pool eaten by a slow dependency · a slow leak     │
 │ 11  │ middleware        │ wrong ORDER → credentials in your logs · no request id, so nothing is ever traceable             │
 │ 12  │ controller        │ business logic and database queries living here, untestable and unreusable                       │
 │ 13  │ authentication    │ a long-lived stateless JWT → logout does not work · a FAST password hash → a leak is a takeover  │
 │ 14  │ authorisation     │ ★★★ CHANGE THE ID IN THE URL AND GET SOMEONE ELSE'S DATA                                       │
 │     │                   │     the most common serious application flaw in existence, and the easiest to test for           │
 │ 15  │ validation        │ trusting a client-sent PRICE or ROLE · SQL built by concatenation · SSRF · a file trusted by     │
 │     │                   │ its extension                                                                                    │
 │ 16  │ business logic    │ ★ NO TIMEOUT on a downstream call → its bad day becomes your outage                             │
 │     │                   │   · a multi-step process with no compensation, leaving records half-finished                     │
 │ 17  │ cache             │ ★★ A CACHE KEY WITHOUT THE USER OR TENANT — a data leak dressed as an optimisation             │
 │     │                   │    · caching used to hide a query that simply needed an index                                    │
 │ 18  │ ORM               │ ★★ THE N+1 QUERY: one query, then one per row. 400 round trips for a single page.               │
 │ 19  │ database          │ a missing index turning 40 ms into 4 seconds · connection pool exhausted, so a healthy database  │
 │     │                   │ looks down · a long transaction holding locks and blocking everything behind it                  │
 │ 20  │ THE FLOOR         │ CPU credits · disk IOPS · memory limits — throttling that LOOKS EXACTLY LIKE slow code          │
 │ 21  │ queue / worker    │ ★★★ NOT IDEMPOTENT → the customer is charged twice                                             │
 │     │                   │     · no dead letter queue → failures vanish silently for months                                │
 │     │                   │     · visibility timeout shorter than the job → two workers do the same work                     │
 │ 22  │ the response      │ a stack trace or internal version returned · 200 OK with an error inside, so nothing alerts      │
 │     │                   │ · an unpaginated list that grows with your success                                               │
 │ 23  │ egress            │ large files proxied through your application · full egress paid because nothing caches           │
 │ 24  │ the render        │ a 45 ms API inside a 4-second page · unsized images · one blocking third-party script            │
 └─────┴───────────────────┴──────────────────────────────────────────────────────────────────────────────────────────────────┘

 ╔══════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════╗
 ║  AND ACROSS ALL OF THEM — the ones that are not at any single station                                                        ║
 ║                                                                                                                             ║
 ║     a destructive migration shipped with the code that needs it        ← rollback becomes impossible                         ║
 ║     one leaked long-lived credential                                   ← identity is the perimeter; it just fell            ║
 ║     backups in the same account, never once restored                   ← replication copies destruction faithfully          ║
 ║     alerts nobody reads                                                ← worse than none: it manufactures confidence        ║
 ║     a rollback path nobody has ever used                               ← its first use should not be during an outage       ║
 ╠══════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════╣
 ║  ★ THE FIVE THAT CAUSE MOST REAL DAMAGE — audit these, this week, in this order                                             ║
 ║                                                                                                                             ║
 ║     1.  station 14   missing authorisation           change an id in a URL. Does it work? Go and try it now.                ║
 ║     2.  station 17   cache key without a tenant      read one cache key out loud. Whose data can it return?                 ║
 ║     3.  station 21   a non-idempotent money handler  what happens if this message arrives twice?                            ║
 ║     4.  station 18   an N+1 query                    count the queries on your busiest page.                                ║
 ║     5.  station 08   a health check that is too deep does it fail when the database does? Then it can remove EVERYTHING.    ║
 ║                                                                                                                             ║
 ║  None of these are exotic. All five are cheap to prevent and expensive to discover in production.                            ║
 ╚══════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════╝
```

---

## The whole thing in six sentences

```text
 1.  A click becomes a network request, and DNS, TCP and TLS all happen before one byte of it reaches you.
 2.  It should be answered as EARLY as possible — at the edge, or in a cache — because everything after that costs money.
 3.  Your application asks three questions in order: who are you, may you, is this valid — and only then does your product.
 4.  The database is almost always the real bottleneck, standing on a floor of CPU, RAM, disk and kernel.
 5.  Anything slow does not belong in the request at all — it goes on a queue, and then it must be idempotent.
 6.  All of it is wrapped in four things that are not stations: identity, observability, reliability, cost.
```

---

## Personal Notes

*Fill this in yourself.*

- **Which poster I printed, and where it is on the wall:**
- **The station where my system is weakest:**
- **Which of the five ★ hazards exist in my system right now:**
- **My own version of POSTER 1, drawn from memory:** → [Diagrams](../17%20-%20Personal%20Knowledge%20Base/Diagrams.md)

---

## Workbook Exercise

- [ ] Print POSTER 1 and POSTER 5. Put them side by side where you work.
- [ ] Cover the labels on POSTER 1 and redraw it from memory. Where you stop is what to study next.
- [ ] Walk one real request in your own system across all 19 outbound stations. Mark the ones you do not have.
- [ ] Go through the five ★ hazards on POSTER 5. Test each one against your system rather than reasoning about it.
- [ ] Find one thing answered at station 19 that could be answered at 06 or 17.
- [ ] Find one thing in your request path that belongs at station 21.
