# Uber Architecture

> **In one line —** a real-time matching problem where geography is the shard key; and the interesting parts are that location is write-heavy, matching is a distributed lock, and money must be exactly right while everything else can be approximate.

| | |
|---|---|
| **Category** | Case Study *(system design)* |
| **Architectural Layer** | Whole-system |
| **Related notes** | [Message Queues](../10%20-%20Distributed%20Systems/Message%20Queues.md) · [WebSockets](../04%20-%20Networking%20and%20Internet/WebSockets.md) · [Redis](../08%20-%20Databases%20and%20Data/Redis.md) · [Distributed Systems Fundamentals](../10%20-%20Distributed%20Systems/Distributed%20Systems%20Fundamentals.md) · [Database Sharding](../08%20-%20Databases%20and%20Data/Database%20Sharding.md) · [Idempotency](../07%20-%20Backend%20Design%20Patterns/Idempotency.md) |

---

## 1. The System in One Page

*What are we designing?*

```text
Match a rider who wants a trip with a nearby driver who will take it,
then track and bill the trip.

    LOCATION    drivers report position every few seconds — CONSTANTLY
    REQUEST     a rider asks for a trip from A to B
    MATCH       find nearby available drivers, offer, get an acceptance
    TRACK       both parties see live position until the trip ends
    PRICE       estimate up front, charge at the end
    PAY         exactly once, correctly, with a receipt
```

> [!IMPORTANT]
> **The defining characteristic is that this is a write-heavy real-time system, which is unusual.** Most consumer systems are read-heavy and cacheable. Here, millions of drivers each emit a location update every few seconds — an enormous, continuous, unavoidable write load whose data is worthless within about ten seconds. That single property drives most of the architecture: it forces a purpose-built geospatial store rather than a general database, and it makes "keep only the current state" a legitimate design.

---

## 2. Requirements and constraints

```text
SCALE (orders of magnitude, publicly discussed)
    millions of active drivers, tens of millions of daily trips
    location updates: MILLIONS PER SECOND at peak
    thousands of microservices
    hundreds of cities, each with its own rules, currency and regulations

CONSTRAINTS
    matching must complete in a few SECONDS — a rider will not wait
    a driver must never be assigned two trips at once
    payment must be exactly-once, and reconcilable
    the mobile network is unreliable; clients drop and reconnect constantly
    physical-world consequences: a wrong match wastes real time and fuel
```

---

## 3. Geography is the natural shard key

```text
A trip in Lisbon has NOTHING to do with a trip in Toronto.
    ↓
So the system partitions by geography, almost for free:
    matching, pricing, supply and demand are all CITY-LOCAL
    ↓
    → shard by city or region
    → deploy regionally, close to the users
    → a failure in one city does not affect another
    → regulations and pricing rules are naturally per-region
```

> [!TIP]
> **When a domain has a natural partition key, the sharding conversation stops being frightening.** Multi-tenant SaaS has tenant id; Uber has geography. The painful sharding projects are the ones where every query needs data from every shard. Before designing anything complex, look for the key along which your workload is already independent — if one exists, use it early, because retrofitting it later is the expensive path.

---

## 4. The geospatial index

```text
THE QUESTION       "which available drivers are within 3 km of this point?"
    ↓
A relational database with lat/long columns cannot answer this fast enough
at millions of writes per second.
```

```text
THE TECHNIQUE — divide the world into cells
    GEOHASH        recursive rectangles; a string prefix = a region
    S2 (Google)    spherical cells, hierarchical
    H3 (Uber's)    HEXAGONAL cells — Uber built and open-sourced this
    ↓
Store: cell id → set of drivers currently in it
Query: my cell + its neighbours → a small candidate list
```

> [!TIP]
> **Hexagons beat squares for this problem because every neighbour is equidistant.** With square cells, diagonal neighbours are further away than edge neighbours, so "expand the search radius" is lumpy and biased. Hexagons have six uniform neighbours, which makes ring-based expansion — search my cell, then ring 1, then ring 2 — clean and consistent. H3 is genuinely useful outside ride-hailing: delivery zones, coverage analysis, any spatial aggregation.

---

## 5. Location updates: the write firehose

```text
millions of drivers × one update every 4 seconds = millions of writes/second
    ↓
PROPERTIES OF THIS DATA
    it is stale within ~10 seconds
    only the LATEST value matters for matching
    the history matters for billing and analytics — but not in real time
```

```text
SO SPLIT IT
    HOT PATH   → in-memory geospatial index (Redis / a purpose-built service)
                 overwrite in place; no durability needed
    COLD PATH  → Kafka → object storage → batch processing
                 the full track, for pricing disputes, ETAs, analytics
```

> [!IMPORTANT]
> **"Only the current value matters" is a licence to discard durability on the hot path, and it is worth looking for in your own systems.** Trying to durably store every location update in a transactional database is the mistake that makes this problem look impossible. Separating the ephemeral current state from the durable event log is what makes it tractable — and the same split applies to presence, cursors, sensor readings and live dashboards.

---

## 6. Matching: a distributed lock in disguise

```text
Rider requests a trip
    ↓
Query the geospatial index → 20 nearby available drivers
    ↓
RANK them: ETA, direction of travel, driver rating, acceptance rate,
           vehicle type, fairness
    ↓
OFFER to the best candidate — with a short timeout (~15 s)
    ↓
Accepted?  → assign
Declined / timed out?  → next candidate
```

> [!CAUTION]
> **The hard requirement is that one driver cannot be offered two trips simultaneously, and this makes matching a mutual-exclusion problem, not a search problem.** Two riders three streets apart can both see the same driver as their best candidate at the same instant. The fix is to atomically claim the driver — a compare-and-set in Redis, or a conditional database update — *before* sending the offer, and release the claim if the offer is declined or times out. Systems that skip this produce the double-assignment bug, which is very visible to real people standing on real pavements.

---

## 7. Real-time tracking

```text
Rider and driver both need continuous position updates during a trip.
    ↓
NOT polling — millions of clients polling every second is wasteful
    ↓
PERSISTENT CONNECTIONS (WebSocket or similar)
    driver → gateway → matching / trip service → rider
    ↓
CONSEQUENCES OF LONG-LIVED CONNECTIONS
    the gateway holds millions of open sockets
    a deployment must not disconnect everyone at once
    mobile networks drop constantly → reconnection must be cheap and stateless
    load balancing does not rebalance existing connections
```

> [!TIP]
> **Long-lived connections change your operational model, and the changes are easy to underestimate.** New instances receive no traffic until clients reconnect, so scaling out does not relieve load; deployments need staged connection draining; and connection state must live outside the gateway so any instance can serve a reconnecting client. See [WebSockets](../04%20-%20Networking%20and%20Internet/WebSockets.md) and [Load Balancing](../14%20-%20Scalability%20and%20Reliability/Load%20Balancing.md).

---

## 8. Two consistency regimes in one system

```text
EVENTUALLY CONSISTENT — and that is fine
    driver location on the map (a second stale is invisible)
    ETA estimates
    surge multipliers
    driver ratings and aggregate statistics

STRONGLY CONSISTENT — non-negotiable
    trip state machine (requested → accepted → started → completed)
    "this driver is assigned to exactly one trip"
    PAYMENTS — exactly once, reconcilable, auditable
    regulatory records
```

> [!IMPORTANT]
> **The skill on display is drawing this line deliberately rather than applying one consistency model everywhere.** Strong consistency for everything would make the location firehose impossible; eventual consistency for everything would double-charge customers. Most real systems have both kinds of data, and the failure mode is not choosing badly — it is never asking the question, and defaulting the whole system to whatever the database does.

---

## 9. Money: exactly-once in a world of retries

```text
The trip ends. Charge the rider.
    ↓
The mobile client retries on a dropped connection.
The payment service times out but MAY have succeeded.
The queue delivers the message twice.
    ↓
WITHOUT IDEMPOTENCY: the customer is charged twice.
```

```text
THE MECHANISM
    an IDEMPOTENCY KEY per charge attempt (usually the trip id)
    the payment service records the key with the outcome
    a repeat with the same key returns the ORIGINAL result — no new charge
    ↓
plus a LEDGER: append-only, double-entry, reconciled against the provider
```

> [!CAUTION]
> **In any distributed system, "did the payment succeed?" is sometimes genuinely unanswerable at the moment you ask.** A timeout tells you nothing about whether the other side committed. Idempotency keys are what convert that ambiguity from a correctness bug into a retryable operation, and a ledger is what lets you prove afterwards what happened. See [Idempotency](../07%20-%20Backend%20Design%20Patterns/Idempotency.md).

---

## 10. Surge pricing

```text
Per geographic cell, continuously:
    open requests  vs  available drivers
    ↓
demand >> supply → raise the multiplier
    ↓
EFFECTS
    some riders wait or decline      → demand falls
    more drivers move to the area    → supply rises
    ↓
a feedback loop that clears the market, per cell, in near real time
```

> [!TIP]
> **This is a good example of computing a value from a stream rather than querying a database.** Surge is derived from a rolling window of events per cell — the kind of aggregation a stream processor does well and a transactional database does badly. Whenever you find yourself running heavy `GROUP BY` queries over recent events on a repeating timer, that is a candidate for a streaming aggregation instead.

---

## 11. Architecture diagram

```text
   driver app                                         rider app
       │ location every ~4 s                              │ trip request
       ▼                                                   ▼
   ┌───────────────── EDGE / API GATEWAY ──────────────────────┐
   │  persistent connections (millions) · auth · rate limits    │
   └────┬──────────────────────────────────────────┬───────────┘
        │                                            │
   ┌────▼──────────────────┐              ┌─────────▼──────────────┐
   │ LOCATION SERVICE       │              │ MATCHING SERVICE        │
   │  in-memory geo index   │◄─────query───┤  rank candidates        │
   │  H3 cell → drivers     │              │  ATOMIC CLAIM on driver │
   │  overwrite in place    │              │  offer with timeout     │
   │  (no durability)       │              └─────────┬──────────────┘
   └────┬──────────────────┘                        │
        │ full track                                 ▼
        ▼                                  ┌──────────────────────┐
   ┌──────────┐                            │ TRIP SERVICE          │
   │  KAFKA   │───► stream processing ───► │  state machine        │
   │          │     (surge, ETA, ML)       │  STRONGLY CONSISTENT  │
   └────┬─────┘                            └──────────┬───────────┘
        │                                              │ trip completed
        ▼                                              ▼
   object storage + warehouse            ┌──────────────────────────┐
   (analytics, disputes, pricing)        │ PAYMENT SERVICE           │
                                          │  idempotency keys         │
                                          │  append-only LEDGER       │
                                          └──────────────────────────┘

   SHARDED BY GEOGRAPHY throughout — a city is an independent unit
```

---

## 12. Complete request flow

```text
─────────────── constantly, in the background ───────────────
Driver app sends position every ~4 s over a persistent connection
    ↓
Location service computes the H3 cell and overwrites the driver's entry
    ↓
The same update is also published to Kafka for the durable track
    ↓
─────────────── a rider requests a trip ───────────────
Request arrives; the rider's city determines the shard
    ↓
Geospatial query: my H3 cell + surrounding rings → 20 available drivers
    ↓
Ranked by ETA, heading, rating, acceptance history, vehicle type
    ↓
ATOMIC CLAIM on the top driver in Redis (compare-and-set)
    → a rider two streets away, matching at the same instant, gets the next one
    ↓
Offer pushed to the driver's app, with a 15-second timeout
    ↓
Driver declines → claim released → offer goes to candidate 2
    ↓
Candidate 2 accepts → the claim becomes an assignment
    ↓
Trip state machine: requested → ACCEPTED  (strongly consistent write)
    ↓
─────────────── during the trip ───────────────
Driver location flows: driver → gateway → trip service → rider's app
    ↓
Rider sees the car move; a second of staleness is invisible and acceptable
    ↓
The route is accumulated in Kafka for distance, duration and disputes
    ↓
─────────────── the trip ends ───────────────
Trip state → COMPLETED (strongly consistent)
    ↓
Fare computed from the durable track, not the ephemeral index
    ↓
Charge submitted with an idempotency key = trip id
    ↓
The mobile client's connection drops; it retries on reconnect
    ↓
Same idempotency key → the original result is returned; NO second charge
    ↓
Ledger entries written; receipt emailed asynchronously
    ↓
─────────────── surge, meanwhile ───────────────
Stream processing sees demand outstripping supply in three adjacent cells
    ↓
Multiplier raised; drivers nearby are notified; some riders defer
    ↓
The imbalance clears within minutes
    ↓
─────────────── a failure ───────────────
The location service loses a node — the in-memory index for those cells is gone
    ↓
Drivers report again within 4 seconds → the index REBUILDS ITSELF
    ↓
Matching in those cells is degraded for a few seconds, then normal
    ↓
Nothing was lost, because nothing durable was being kept there
```

---

## 13. Key design decisions, and what they cost

| Decision | Bought | Cost |
|---|---|---|
| **Shard by geography** | Natural independence, regional isolation | Cross-city features are awkward |
| **In-memory geo index** | Millions of writes/second | No durability — accepted deliberately |
| **H3 hexagonal cells** | Uniform neighbour distance, clean expansion | A new spatial vocabulary to learn |
| **Kafka for the durable track** | Analytics, disputes, ML, replay | Two paths for the same data |
| **Atomic driver claim** | No double assignment | Coordination on the hot path |
| **Two consistency regimes** | Scale where possible, correctness where required | Every dataset needs a decision |
| **Idempotency + ledger** | Exactly-once billing under retries | Extra state and reconciliation work |
| **Persistent connections** | Live tracking without polling | Millions of sockets to operate |

---

## 14. What transfers to a normal system

```text
DIRECTLY APPLICABLE
    ✓ find your natural partition key (tenant, region, account) and use it EARLY
    ✓ split ephemeral current state from the durable event log
    ✓ idempotency keys on anything that moves money or sends messages
    ✓ decide consistency PER DATASET, not per system
    ✓ compute aggregates from a stream instead of repeated GROUP BY queries
    ✓ atomically claim a resource before offering it, if double-booking is possible
    ✓ H3 or S2 for ANY spatial query — delivery zones, coverage, catchments

DO NOT COPY WITHOUT THE SCALE
    ✗ thousands of microservices
    ✗ a bespoke in-memory geospatial service — PostGIS handles a great deal
    ✗ Kafka, if a queue would do
```

> [!TIP]
> **For most applications with location features, PostGIS is the correct answer and this entire note is context.** A spatial index in PostgreSQL comfortably serves thousands of queries per second over millions of rows — the specialised architecture here exists because of the write rate, not the query. Reach for the specialised design when you can name the number that rules out the simple one.

---

## 15. Security Considerations

> [!CAUTION]
> **Location data is among the most sensitive categories of personal data there is: it reveals home, work, medical visits and relationships.** This system's security design is dominated by that fact, and any application collecting location inherits the same obligation. Precision, retention and access are all design decisions with real consequences.

- **Minimise precision and retention** — keep exact tracks only as long as disputes require
- **Show approximate location before a match is confirmed**, exact only during the trip
- **Strict access control and audit on location queries** — internal misuse of location data has caused real scandals in this industry
- **Payment data isolated**, with PCI scope kept as small as possible
- **Trust nothing from the client** — a spoofed GPS position or a client-supplied fare is an attack, so fares must be computed server-side
- **Rate limit account creation and trip requests**; fake accounts and collusion fraud are constant
- **Per-city regulatory requirements** for data residency and retention
- **Persistent connections must be authenticated on reconnect**, not just at first connect

---

## 16. Mental Model

> [!NOTE]
> **Uber is a taxi dispatcher with a whiteboard, a filing cabinet and a cash book.**
>
> The whiteboard shows where every car is *right now*; it is wiped and redrawn constantly, and if it were destroyed the drivers would radio in and it would be rebuilt in seconds — nothing on it is worth keeping. The filing cabinet holds the complete record of every journey, written once, never altered, and used for billing and disputes. The cash book must balance exactly, every day, no matter how many times a radio call was repeated. Three different kinds of data, three different guarantees, one system.

---

## 17. Key Takeaway

> [!IMPORTANT]
> Uber's transferable lessons are: shard along your domain's natural boundary, separate ephemeral current state from the durable event log, use idempotency keys for anything involving money, and decide consistency requirements per dataset rather than for the system as a whole.

---

## 18. Common Mistakes When Copying This

- **Durably writing every location update** to a transactional database
- **Lat/long columns with no spatial index**, then wondering why proximity queries are slow
- **No atomic claim before offering**, producing double assignments
- **Polling instead of persistent connections**, or persistent connections without a plan for deployments
- **One consistency model for the whole system** — either too weak for payments or too strong for location
- **No idempotency keys**, so a retry becomes a duplicate charge
- **Trusting client-supplied location or fare**
- **Building a bespoke geospatial service** when PostGIS would serve the actual load
- **Sharding by something unnatural** when a domain boundary was available
- **Retaining precise location history indefinitely**, creating a liability with no product purpose

---

## 19. Open Source Technologies

- **H3** — Uber's hexagonal geospatial indexing library; genuinely useful and widely adopted
- **S2** — Google's spherical cell library; the main alternative
- **PostGIS** — the right answer for most applications with spatial queries
- **Redis** — geospatial commands, atomic compare-and-set for claims, ephemeral state
- **Apache Kafka** — the durable event log and the backbone for stream processing
- **Apache Flink**, **Kafka Streams** — surge, ETA and real-time aggregation
- **Cadence / Temporal** — Uber-originated workflow engines; excellent for long-running, retryable business processes
- **Ringpop**, **TChannel** — Uber's older service-mesh-era OSS, largely historical now
- **Envoy** — the modern layer for persistent connections and service traffic

---

## 20. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 21. Workbook Exercise

- [ ] Identify your system's natural partition key. If you have one, are you using it?
- [ ] Find a dataset you store durably where only the latest value actually matters.
- [ ] Check whether any operation that charges money or sends a message lacks an idempotency key.
- [ ] List your datasets and mark each strongly or eventually consistent. Note any you have never considered.
- [ ] Find a repeated aggregation query that could be a streaming computation instead.
- [ ] If you handle location, check your retention period and ask what it is for.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
persistent connections → in-memory H3 geo index (ephemeral) → matching with atomic claims
                      → Kafka (durable track) → stream processing + warehouse
                      → trip state machine (strong) → payments (idempotent + ledger)
```

## 2. Request Flow

```text
Input       a constant firehose of driver locations, plus occasional trip requests
    ↓
Processing  cell-based proximity search, ranked offers with atomic exclusion, a strict state machine
    ↓
Output      a match within seconds, live tracking, and a charge that happens exactly once
```

## 3. Real-World Usage

Ride-hailing, delivery, logistics and field-service platforms all share this shape, and the parts that generalise furthest are the least glamorous: geographic sharding, ephemeral-versus-durable data separation, idempotent payments, and per-dataset consistency decisions. H3 escaped the domain entirely and is now used wherever spatial aggregation is needed.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A geographically sharded real-time matching system with strict money handling |
| **Why is it built this way?** | Because location is a write firehose whose value expires in seconds, while payment must be exact |
| **Where does it belong?** | As a reference for real-time matching and mixed-consistency design |
| **What should I take from it?** | Natural sharding, ephemeral versus durable data, idempotency, and atomic claims |
