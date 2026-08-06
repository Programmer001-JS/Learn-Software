# Netflix Architecture

> **In one line —** the video bytes never touch Netflix's cloud; everything else does — and understanding that split explains the entire architecture.

| | |
|---|---|
| **Category** | Case Study *(system design)* |
| **Architectural Layer** | Whole-system |
| **Related notes** | [Video Processing Platform](Video%20Processing%20Platform.md) · [Internet Fundamentals](../04%20-%20Networking%20and%20Internet/Internet%20Fundamentals.md) · [Microservices](../10%20-%20Distributed%20Systems/Microservices.md) · [High Availability](../14%20-%20Scalability%20and%20Reliability/High%20Availability.md) · [Cloud Fundamentals](../12%20-%20Cloud%20Architecture/Cloud%20Fundamentals.md) · [Cache](../08%20-%20Databases%20and%20Data/Cache.md) |

---

## 1. The System in One Page

*What are we designing?*

```text
Hundreds of millions of subscribers, in ~190 countries, streaming video.

    BROWSE      personalised catalogue, thumbnails, rows of recommendations
    PLAY        start in under 2 seconds, adapt to bandwidth, never buffer
    ENCODE      one uploaded master → hundreds of encoded variants
    DELIVER     tens of terabits per second at peak
    RECOMMEND   what to show, per user, per device, per row
```

> [!IMPORTANT]
> **The architectural insight that makes Netflix comprehensible: it is two systems, not one.** The **control plane** — sign-in, search, recommendations, playback authorisation — runs as microservices in AWS. The **data plane** — the actual video bytes, which are essentially all of the traffic — is served from Netflix's own CDN, Open Connect, installed inside internet service providers. Splitting a system along "the enormous, cacheable, immutable part" versus "the small, dynamic, personalised part" is the transferable lesson here.

---

## 2. Requirements and constraints

```text
SCALE (publicly discussed, orders of magnitude)
    hundreds of millions of subscribers
    peak traffic measured in TENS OF TERABITS per second
    a very large share of downstream internet traffic in some markets
    thousands of microservices
    billions of playback and interaction events per day

CONSTRAINTS
    video files are enormous and IMMUTABLE once encoded
    playback must start fast and never stall
    availability matters more than perfect personalisation
    global, with wildly varying network quality
    regional content licensing
```

---

## 3. The two-plane architecture

```text
   ┌──────────────── CONTROL PLANE (AWS) ─────────────────┐
   │  API gateway (Zuul) → thousands of microservices      │
   │  sign-in · profiles · search · recommendations        │
   │  playback authorisation · billing · A/B experiments   │
   │  Cassandra · EVCache · Kafka · Spark                  │
   │  ~small bytes, HIGHLY dynamic, personalised            │
   └────────────────────┬─────────────────────────────────┘
                        │  returns a manifest: URLs + tokens
                        ▼
   ┌──────────── DATA PLANE (Open Connect) ───────────────┐
   │  Netflix-built appliances INSIDE ISP networks          │
   │  and at internet exchanges                             │
   │  video segments, immutable, pre-positioned overnight   │
   │  ~ALL the bytes, ZERO personalisation                  │
   └───────────────────────────────────────────────────────┘
```

> [!TIP]
> **Notice what the control plane returns: not video, but a list of URLs.** The client asks "may I watch this, and where?" — and from that point on, Netflix's cloud is not in the path. This is the same pattern as a presigned S3 URL, at planetary scale: authorise centrally, serve the bytes from wherever is closest.

---

## 4. Open Connect: the part that is genuinely unusual

```text
Netflix built its own CDN and gives the hardware to ISPs for free.

    WHY IT WORKS
        video is IMMUTABLE and PREDICTABLE
        → a small number of titles account for most viewing
        → so the popular catalogue can be pre-loaded, overnight,
          during the ISP's quiet hours
        ↓
        at peak, the bytes travel from a box inside the ISP
        to the subscriber — often never crossing the public internet
```

```text
THE ECONOMICS
    the ISP saves enormous transit cost
    Netflix saves egress cost and gets better quality
    the subscriber gets lower latency and fewer stalls
```

> [!IMPORTANT]
> **Pre-positioning only works because the content is immutable and its popularity is predictable.** A general CDN caches reactively, on a miss. Netflix can fill caches proactively, hours in advance, because it knows what will be watched and the file will never change. Ask of your own system: *what part of my data is immutable and predictable?* — that is the part that can be pushed to the edge rather than pulled through it.

---

## 5. Encoding: one master, hundreds of variants

```text
studio master (very large, high quality)
    ↓
split into SHOTS, encoded in parallel across thousands of machines
    ↓
PER-TITLE / PER-SHOT ENCODING
    a simple animation needs far fewer bits than a dark action scene
    → the ladder is optimised per title, not fixed globally
    ↓
hundreds of outputs: codecs × resolutions × bitrates × audio × subtitles
    ↓
validated, packaged, then DISTRIBUTED to Open Connect appliances
```

> [!TIP]
> **Per-title encoding is a good example of replacing a fixed policy with a measured one.** A one-size-fits-all bitrate ladder wastes bandwidth on simple content and under-serves complex content. Analysing each title costs compute once and saves bandwidth on every subsequent stream — an excellent trade when a title is watched millions of times. See [Video Processing Platform](Video%20Processing%20Platform.md).

---

## 6. Adaptive bitrate streaming

```text
The video is cut into SEGMENTS (a few seconds each), at many bitrates.
    ↓
The CLIENT decides, segment by segment:
    buffer healthy + bandwidth good   → request a higher bitrate
    buffer draining                   → drop down immediately
    ↓
Result: quality degrades instead of playback stopping
```

> [!IMPORTANT]
> **The client, not the server, makes the quality decision — and that is what makes the system scale.** The server just hosts static files at various bitrates; all the adaptation logic lives in the player, where the actual network conditions are visible. No server-side session, no coordination, no state. The architectural principle is general: **push decisions to whoever has the information, and keep the serving layer dumb and cacheable.**

---

## 7. Resilience: designed for constant failure

```text
THE PRACTICES NETFLIX POPULARISED
    CHAOS ENGINEERING     deliberately kill instances in production
                          → failover paths stay honest because they are used
    CIRCUIT BREAKERS      (Hystrix, then resilience4j) stop calling a sick service
    BULKHEADS             separate thread pools per dependency
    FALLBACKS             a degraded response beats an error
    REGIONAL EVACUATION   shift all traffic out of an AWS region
    STATELESS SERVICES    any instance can be destroyed at any moment
```

```text
GRACEFUL DEGRADATION IN PRACTICE
    recommendation service down  → show a generic popular list, not an error
    personalised artwork down    → show default artwork
    "continue watching" down     → the rest of the page still works
    → the user notices something is less good; they can still watch
```

> [!IMPORTANT]
> **The transferable idea is not "run Chaos Monkey" — it is that every dependency has a defined, tested behaviour when it fails.** For each service call, someone decided in advance what the user sees if it times out. That is a design activity, not an infrastructure one, and it is the reason a failure in one of thousands of services does not produce a blank page.

---

## 8. Data: purpose-built stores, not one database

```text
CASSANDRA        viewing history, user state
                 → multi-region, write-heavy, eventually consistent
EVCACHE          a memcached-based caching tier, replicated across zones
                 → most reads never reach a database
KAFKA            the event backbone: playback events, telemetry, experiments
S3 + Spark/Flink batch and stream processing for recommendations
ELASTICSEARCH    search and operational insight
```

> [!TIP]
> **Eventual consistency is a deliberate product decision here, not a compromise.** If your viewing position syncs across devices a few seconds late, nobody minds — and accepting that unlocks multi-region availability that strong consistency would forbid. The skill is identifying which data genuinely needs strong consistency (billing, entitlements) and which does not (viewing progress, recommendations).

---

## 9. Microservices: the honest assessment

```text
WHAT NETFLIX GOT FROM THEM
    thousands of engineers deploying independently
    per-service scaling and technology choice
    failure isolation, with fallbacks

WHAT THEY COST
    a service mesh, discovery, distributed tracing, config management
    an entire platform organisation to build and run the tooling
    debugging across dozens of hops
    an eventually-consistent data model
```

> [!CAUTION]
> **Netflix adopted microservices because it had thousands of engineers, and the tooling it built was necessary to make them survivable — not because microservices are inherently better.** Copying this architecture with twenty engineers imports every cost and none of the organisational benefit. The correct lesson from Netflix is the two-plane split, the fallback discipline, and the client-side adaptation — not the service count. See [Microservices](../10%20-%20Distributed%20Systems/Microservices.md).

---

## 10. Architecture diagram

```text
                            client (TV, phone, browser)
                                    │
                    ┌───────────────┴────────────────┐
                    │ 1. control requests            │ 2. video bytes
                    ▼                                 ▼
   ┌──────── AWS (multi-region) ────────┐   ┌──── OPEN CONNECT ────────┐
   │  API gateway / edge                 │   │  appliance INSIDE the ISP │
   │        │                             │   │  pre-filled overnight     │
   │  ┌─────▼──────────────────────────┐ │   │  immutable segments        │
   │  │ playback authorisation          │ │   │                            │
   │  │ recommendations · search        │ │   │  → the vast majority of    │
   │  │ profiles · billing · experiments│ │   │    all bytes served here   │
   │  │  (thousands of services, each   │ │   └────────────▲──────────────┘
   │  │   with timeouts + fallbacks)    │ │                │
   │  └─────┬──────────────────────────┘ │                │
   │        │                             │   fill during off-peak hours
   │  EVCache ──► Cassandra               │                │
   │        │                             │                │
   │  Kafka ──► Spark/Flink ──► S3 ───────┼────────────────┘
   │            (recommendations,          │   encoded outputs distributed
   │             telemetry, A/B)           │
   └───────────────────────────────────────┘

   returns: a MANIFEST (segment URLs + tokens) — never the video itself
```

---

## 11. Complete request flow

```text
User opens Netflix on a TV
    ↓
Client authenticates; the edge routes to the nearest healthy AWS region
    ↓
Home page assembled from MANY service calls, in parallel:
    rows, artwork, "continue watching", search suggestions
    ↓
The recommendation service is slow → its 80 ms timeout fires
    → FALLBACK: a generic popular-titles row
    → the page renders fully; the user notices nothing specific
    ↓
Most of these calls are served from EVCache, not a database
    ↓
─────────────── the user presses play ───────────────
Client requests playback for title X on this device, in this country
    ↓
Control plane checks: entitlement, regional licensing, device capabilities (DRM,
codec support), and the account's simultaneous-stream limit
    ↓
Returns a MANIFEST: segment URLs on the NEAREST Open Connect appliance,
plus DRM licence information — and now steps aside
    ↓
─────────────── streaming ───────────────
Client fetches segment 1 at a conservative bitrate → playback starts <2 s
    ↓
Buffer fills; measured bandwidth is good → step up to a higher bitrate
    ↓
Bytes come from a box inside the user's ISP — often never crossing the public internet
    ↓
Someone else in the house starts a video call; bandwidth halves
    ↓
Client detects the buffer draining → drops to a lower bitrate mid-stream
    ↓
Quality degrades briefly. Playback does not stop. THIS IS THE DESIGN.
    ↓
─────────────── telemetry ───────────────
Playback events stream to Kafka: bitrate changes, stalls, position, completion
    ↓
Feeds recommendations, quality-of-experience dashboards, and A/B analysis
    ↓
─────────────── an AWS region degrades ───────────────
Control-plane traffic is evacuated to another region
    ↓
Sessions already streaming CONTINUE, because the bytes were never coming from AWS
    ↓
New playback starts are slower to authorise but still work
    ↓
The two-plane split means a cloud outage degrades browsing, not watching
```

---

## 12. Key design decisions, and what they cost

| Decision | Bought | Cost |
|---|---|---|
| **Own CDN inside ISPs** | Enormous egress savings, better quality | Hardware, logistics, ISP relationships |
| **Two-plane split** | Cloud outages do not stop playback | Two operational domains |
| **Client-side ABR** | Serving layer stays dumb and cacheable | Complex player logic on many devices |
| **Per-title encoding** | Less bandwidth for the same quality | Large one-off compute per title |
| **Microservices** | Thousands of engineers ship independently | An entire platform organisation |
| **Eventual consistency** | Multi-region availability | Occasional visible staleness |
| **Chaos engineering** | Failover paths that actually work | Deliberate production risk |
| **Fallbacks everywhere** | Degradation instead of errors | Every call needs a designed failure mode |

---

## 13. What transfers to a normal system

```text
DIRECTLY APPLICABLE, at any size
    ✓ separate the enormous cacheable bytes from the small dynamic part
    ✓ serve immutable content from a CDN; keep it out of your application
    ✓ define a fallback for EVERY dependency, and test it
    ✓ timeouts on every call, always
    ✓ cache aggressively in front of databases
    ✓ let clients adapt, rather than negotiating server-side
    ✓ accept eventual consistency where the product allows it

DO NOT COPY WITHOUT NETFLIX'S SCALE
    ✗ thousands of microservices
    ✗ your own CDN hardware
    ✗ a bespoke platform organisation
    ✗ Cassandra, unless you have its specific write pattern
```

> [!TIP]
> **The single most portable idea here is the presigned-URL pattern: authorise centrally, serve the bytes elsewhere.** Any application handling media, documents or large downloads can do exactly this with S3 and a CDN, and it removes the largest source of load from the application tier. That is Netflix's architecture in miniature, available to a team of three.

---

## 14. Security Considerations

> [!CAUTION]
> **Content protection is a hard requirement here in a way it is not for most systems, and it shapes the architecture.** DRM licensing, device attestation and playback tokens exist because studio contracts require them — which means the manifest cannot simply be a public URL. It is a signed, short-lived, device-bound grant.

- **Short-lived signed URLs and tokens** for segments; a leaked manifest expires quickly
- **DRM** (Widevine, PlayReady, FairPlay) with licence servers in the control plane
- **Device attestation**, and per-account concurrent stream limits
- **Regional licensing enforced server-side**, not in the client
- **Open Connect appliances are Netflix-controlled hardware** in third-party networks — a genuinely interesting trust boundary
- **Credential stuffing and account sharing** are the dominant abuse vectors; see [Login System Architecture](Login%20System%20Architecture.md)
- **Billing and entitlement need strong consistency**, unlike viewing history

---

## 15. Mental Model

> [!NOTE]
> **Netflix is a ticket office and a warehouse network, deliberately kept apart.**
>
> The ticket office (AWS) is busy, personalised and clever: it knows who you are, what you are allowed to watch, and what you might enjoy. It hands you a ticket with an address on it. The warehouses (Open Connect) are dumb, enormous, and located as close to you as possible — they hold identical copies of the same boxes, restocked overnight, and they do nothing but hand over the box named on your ticket. If the ticket office has a bad day you may see a duller list of films; if you are already watching one, the warehouse does not care at all.

---

## 16. Key Takeaway

> [!IMPORTANT]
> Netflix's real architecture lesson is the split: keep the huge immutable bytes on a dumb, cacheable delivery layer as close to users as possible, and keep the small dynamic personalised logic in your application — with a designed fallback for every dependency.

---

## 17. Common Mistakes When Copying This

- **Adopting microservices** because Netflix did, without thousands of engineers
- **Serving large media through the application tier** instead of a CDN
- **No fallbacks**, so one slow dependency produces a blank page
- **Reactive caching only**, when your popular content is predictable
- **Server-side quality negotiation** where the client has better information
- **Cassandra chosen for its reputation** rather than for a matching write pattern
- **Eventual consistency applied to billing or entitlements**
- **Chaos engineering introduced** before basic redundancy and rollback exist
- **Building an internal platform** for a team that does not need one

---

## 18. Open Source Technologies

- **Zuul**, **Eureka**, **Ribbon**, **Hystrix** — Netflix's own OSS, historically influential; Hystrix is superseded by **resilience4j**
- **Apache Cassandra** — the multi-region write-heavy store
- **EVCache** — Netflix's memcached-based caching layer
- **Apache Kafka**, **Flink**, **Spark** — the event and processing backbone
- **Chaos Monkey / Simian Army**, **Chaos Mesh**, **LitmusChaos** — deliberate failure injection
- **Spinnaker** — Netflix's multi-cloud progressive delivery tool
- **FFmpeg**, **SVT-AV1**, **x265**, **VMAF** — the encoding toolchain and Netflix's own quality metric
- **Shaka Player**, **dash.js**, **hls.js** — adaptive bitrate players
- **Atlas**, **Conductor** — metrics and workflow orchestration

---

## 19. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 20. Workbook Exercise

- [ ] Identify the "enormous immutable part" of your own system. Is it served from a CDN?
- [ ] Pick one dependency and write down what the user should see if it times out. Then check the code.
- [ ] Find one place where your application proxies bytes it could hand off with a signed URL.
- [ ] List which of your data has a genuine strong-consistency requirement. It is usually less than you think.
- [ ] Draw your system as two planes — control and data. Note anything sitting awkwardly across both.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
control plane (AWS microservices, cached, fallback-everywhere) → issues a manifest
data plane (Open Connect inside ISPs, immutable, pre-filled) → serves all the bytes
```

## 2. Request Flow

```text
Input       "who am I, and may I watch this?"
    ↓
Processing  personalised authorisation in the cloud; bytes served from inside the ISP
    ↓
Output      playback that starts in seconds and degrades in quality rather than stopping
```

## 3. Real-World Usage

Netflix is the most documented large architecture in the industry, and it is most often mis-copied: teams take the microservice count and leave behind the ideas that would actually help them. What generalises is the two-plane split, the discipline of a designed fallback per dependency, client-side adaptation, and pushing immutable data as close to users as possible.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A cloud control plane plus a purpose-built CDN that serves nearly all the traffic |
| **Why is it built this way?** | Because video is enormous, immutable and predictable, while personalisation is not |
| **Where does it belong?** | As a reference for separating cacheable bulk from dynamic logic |
| **What should I take from it?** | The split, the fallbacks, the timeouts — not the service count |
