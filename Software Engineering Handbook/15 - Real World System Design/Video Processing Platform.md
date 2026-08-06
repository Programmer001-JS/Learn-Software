# Video Processing Platform

> **In one line —** the canonical asynchronous pipeline: accept a large file quickly, do expensive work out of band, and never let a user wait for a transcode.

| | |
|---|---|
| **Category** | Case Study *(system design)* |
| **Architectural Layer** | Whole-system |
| **Related notes** | [Netflix Architecture](Netflix%20Architecture.md) · [S3](../12%20-%20Cloud%20Architecture/S3.md) · [Message Queues](../10%20-%20Distributed%20Systems/Message%20Queues.md) · [Background Workers](../10%20-%20Distributed%20Systems/Background%20Workers.md) · [Internet Fundamentals](../04%20-%20Networking%20and%20Internet/Internet%20Fundamentals.md) · [EC2](../12%20-%20Cloud%20Architecture/EC2.md) |

---

## 1. The System in One Page

*What are we designing?*

```text
A user uploads a video. Other users watch it, on any device, at any bandwidth.

    UPLOAD      a 4 GB file, from a flaky connection, resumably
    VALIDATE    is it really a video? is it safe? is it allowed?
    TRANSCODE   one master → many resolutions, bitrates, codecs
    PACKAGE     segment for adaptive streaming; generate thumbnails
    DELIVER     from a CDN, adapting to each viewer's bandwidth
    OBSERVE     progress, failures, cost per minute of video
```

> [!IMPORTANT]
> **This is the reference architecture for every "user uploads something expensive to process" problem.** Change the words and it becomes document conversion, image processing, audio transcription, PDF generation or ML inference on user files. The shape is always identical: **accept fast, queue, process in a fleet you can scale to zero, store the results immutably, and serve them from somewhere cheap.** Video is simply the version where every constraint is exaggerated enough to be obvious.

---

## 2. Requirements and constraints

```text
FUNCTIONAL
    resumable upload of multi-gigabyte files
    several output renditions per input
    thumbnails, preview sprites, subtitles
    visible progress, and a clear failure state
    adaptive playback on phones, browsers and TVs

NON-FUNCTIONAL
    upload must not pass through the application tier
    transcoding is CPU/GPU-heavy and takes MINUTES to HOURS
    load is spiky — 5 uploads one hour, 5,000 the next
    cost per minute of video is the metric that decides the business
    a failed transcode must be retryable without re-uploading
```

---

## 3. The architecture

```text
   browser/app
       │ 1. "I want to upload"
       ▼
   ┌─────────────────┐   2. presigned multipart URL (auth checked here)
   │  API service    │──────────────────────────────┐
   └─────────────────┘                              │
       │ 3. metadata row: status = uploading        │
       ▼                                             ▼
   ┌──────────┐                        ┌──────────────────────────┐
   │ database │                        │  OBJECT STORAGE (S3)      │
   │ jobs +   │                        │  raw/  ← uploaded direct  │
   │ metadata │                        │  renditions/ ← outputs    │
   └──────────┘                        └────────────┬─────────────┘
       ▲                                             │ 4. ObjectCreated event
       │                                             ▼
       │                          ┌──────────────────────────────┐
       │                          │  ORCHESTRATOR                 │
       │  status updates          │  probe → plan the ladder →    │
       └──────────────────────────┤  fan out segment jobs         │
                                  └───────────┬──────────────────┘
                                              ▼
                                  ┌────────────────────────────┐
                                  │  QUEUE (visibility timeout)│
                                  └───────┬───────────┬────────┘
                                          ▼           ▼
                              ┌────────────────┐ ┌────────────────┐
                              │ WORKER (spot)  │ │ WORKER (spot)  │
                              │ FFmpeg         │ │ FFmpeg         │
                              │ autoscaled on  │ │ scale to ZERO  │
                              │ queue depth    │ │ when idle      │
                              └───────┬────────┘ └───────┬────────┘
                                      └────────┬─────────┘
                                               ▼
                                       renditions → S3
                                               │
                                       ┌───────▼────────┐
                                       │  CDN           │ → viewers
                                       └────────────────┘
```

---

## 4. Upload: never through your application

```text
WRONG                                RIGHT
browser → your API → S3              browser asks for a presigned URL
    ↓                                browser uploads DIRECTLY to S3
4 GB through your server                  ↓
memory, timeouts, bandwidth,         your server never sees a byte
one upload can occupy a worker       S3 handles concurrency and scale
```

```text
MULTIPART UPLOAD does the rest
    the file is split into parts, uploaded in PARALLEL
    a failed part is retried alone — not the whole 4 GB
    the upload RESUMES after a dropped connection
    → essential on mobile networks
```

> [!IMPORTANT]
> **Presigned multipart upload is the single most important decision in this system.** It removes bandwidth, memory pressure, request timeouts and a scaling constraint from your application in one step, and it gives users resumability for free. Any system where files pass through the application tier will eventually hit all four problems at once — usually on the day someone uploads something large.

---

## 5. Validation: the upload is untrusted input

```text
BEFORE spending money on transcoding
    verify it IS a video          → probe the container, do not trust the extension
    check duration and resolution  → within plan limits?
    check the codec is supported
    scan for malware
    content moderation, if required
    ↓
FAIL FAST. A rejected file should cost cents, not compute hours.
```

> [!CAUTION]
> **A media file is a program for a decoder, and decoders have a long history of vulnerabilities.** A crafted file can crash, hang or exploit FFmpeg. Treat every upload as hostile: run transcoding in an isolated container with no credentials beyond what the job needs, no network access it does not require, a hard CPU and wall-clock limit, and a read-only filesystem apart from scratch space. This is one of the clearest real cases for stronger isolation than a plain container — a microVM is a reasonable choice here.

---

## 6. Transcoding: the expensive part

```text
ONE INPUT → A LADDER OF OUTPUTS
    1080p @ 5 Mbps · 720p @ 2.5 · 480p @ 1 · 360p @ 0.6 · audio-only
    × codecs (H.264 for compatibility, AV1/HEVC for efficiency)
    × packaging (HLS, DASH)
```

```text
PARALLELISM IS THE WHOLE GAME
    a 2-hour film transcoded serially = many hours
    ↓
    SPLIT BY SEGMENT (or scene), transcode segments IN PARALLEL,
    then concatenate
    ↓
    100 workers → minutes instead of hours
```

> [!TIP]
> **Segment-parallel transcoding is what makes this economically sensible, and it works because video segments are independent at keyframe boundaries.** Split at keyframes, encode each chunk on a separate worker, then stitch. It also makes retries cheap — one failed segment is re-encoded alone rather than restarting a two-hour job. Getting the segment boundaries right is the fiddly part; everything else follows.

---

## 7. Jobs, retries and the state machine

```text
uploading → uploaded → validating → transcoding → packaging → ready
                 │           │            │            │
                 └───────────┴────────────┴────────────┴──► failed
                                                             │
                                                        retryable?
```

```text
THE RULES THAT MAKE IT SURVIVABLE
    workers are IDEMPOTENT — the same job twice produces the same output
    output keys are DETERMINISTIC — derived from input id + rendition
    a queue VISIBILITY TIMEOUT longer than the longest job
    limited retries, then a DEAD LETTER QUEUE with an alarm
    progress reported per segment, so a stuck job is visible
```

> [!CAUTION]
> **A visibility timeout shorter than your longest job is the classic failure of this architecture.** The queue decides the message was lost, redelivers it, and now two workers transcode the same video — doubling cost and racing to write the same output. Either set the timeout above your worst case, or extend it with a heartbeat while working. Deterministic output keys make the race harmless rather than corrupting, which is why they matter.

---

## 8. Spot instances: where the economics live

```text
Transcoding is the ideal spot workload
    interruptible          → one segment re-runs, cheap
    stateless              → nothing lost
    retryable              → the queue redelivers automatically
    ↓
    up to ~90% cheaper than on-demand
```

```text
HANDLING INTERRUPTION
    listen for the termination notice (~2 minutes)
    stop taking new segments; let the current one finish or abandon it
    the queue redelivers what was not acknowledged
```

> [!IMPORTANT]
> **For a compute-dominated pipeline, spot capacity is not an optimisation — it is the difference between a viable and an unviable unit cost.** The engineering required is small: handle the termination notice and make jobs idempotent, both of which you want anyway. Any workload behind a queue with retryable, stateless jobs should be asking why it is not on spot; see [EC2](../12%20-%20Cloud%20Architecture/EC2.md).

---

## 9. Adaptive streaming and delivery

```text
The output is not one file. It is many small SEGMENTS plus a MANIFEST.
    ↓
The manifest lists every available bitrate.
The PLAYER chooses, segment by segment, based on its own measurements.
    ↓
Server-side: purely static files. Cacheable. Dumb. Perfect for a CDN.
```

```text
COST NOTE
    egress from object storage to viewers is the largest recurring bill
    → a CDN in front is cheaper AND faster
    → cache the segments aggressively; they are IMMUTABLE
```

> [!TIP]
> **Because segments never change, they are perfectly cacheable — so put a long cache lifetime on segments and a short one on manifests.** This single split gives you near-total CDN offload for the bytes while keeping the ability to update a rendition ladder. It is the same immutable-content principle that makes Netflix's pre-positioning possible; see [Netflix Architecture](Netflix%20Architecture.md).

---

## 10. Real World Example

- **YouTube, Vimeo, Twitch VOD** — the archetype, with much larger ladders and ML on top.
- **Netflix's encoding pipeline** — per-title ladders and shot-based encoding; see [Netflix Architecture](Netflix%20Architecture.md).
- **Course and webinar platforms** — the same pipeline at a thousandth of the scale, and the architecture barely changes.
- **User-generated content moderation** — the same pipeline with a classification step added.
- **The same shape, different payload** — image resizing, document conversion, audio transcription, batch ML inference.

---

## 11. Communication and Dependencies

- **Object storage** with multipart, presigning, versioning and lifecycle rules; see [S3](../12%20-%20Cloud%20Architecture/S3.md)
- **A queue** with a dead letter queue and a suitable visibility timeout; see [Message Queues](../10%20-%20Distributed%20Systems/Message%20Queues.md)
- **A worker fleet** that scales on queue depth and to zero; see [Background Workers](../10%20-%20Distributed%20Systems/Background%20Workers.md)
- **A relational database** for job state and metadata — the authoritative record
- **A CDN**, without which egress dominates the bill
- **FFmpeg**, and a hard-won familiarity with its flags
- **Monitoring on queue age, failure rate and cost per minute of video**

---

## 12. When To Use / When NOT To Use

> [!TIP]
> Use this pattern for any user-supplied work that takes longer than a request should. If a task exceeds a couple of seconds, it belongs behind a queue — and if it exceeds a minute, it needs progress reporting and a retry story.

> [!CAUTION]
> - **Do not transcode inside a web request** — it will time out, and one upload will occupy a server
> - **Do not use serverless functions** for long transcodes; the execution limit is a wall; see [Lambda](../12%20-%20Cloud%20Architecture/Lambda.md)
> - **Do not build this before checking a managed service** — MediaConvert, Mux or Cloudflare Stream may be cheaper than your engineering time
> - **Do not skip validation** and pay to transcode rubbish
> - **Do not serve segments directly from object storage** at any volume
> - **Do not run FFmpeg on untrusted input without isolation**
> - **Do not store only in object storage** — without a database row, you cannot find or manage anything

---

## 13. Advantages and Disadvantages

**Advantages**
- The user waits seconds, not hours
- Workers scale to zero, so idle costs nothing
- Spot capacity makes the unit economics work
- A failure retries one segment, not the whole job
- Outputs are immutable, so the CDN does almost all the delivery work
- The same architecture handles any expensive asynchronous task

**Disadvantages**
- **Asynchronous means the UI must handle a pending state** — a real product cost
- Many moving parts: queue, orchestrator, workers, storage, CDN
- Debugging a failure spans several systems
- Transcoding is genuinely expensive; cost engineering is permanent
- FFmpeg is powerful, arcane, and a security surface
- Storage grows relentlessly without lifecycle rules

---

## 14. Performance Impact

| Aspect | Impact |
|---|---|
| **Upload** | Bounded by the client's connection, not your servers |
| **Segment-parallel transcode** | Wall-clock scales almost linearly with worker count |
| **GPU encoding** | Much faster, slightly lower quality per bit than a slow CPU preset |
| **Codec choice** | AV1/HEVC save bandwidth, cost far more CPU to encode |
| **Queue-depth autoscaling** | Minutes to add capacity — acceptable here, unlike a web tier |
| **Cold start of workers** | Bake FFmpeg into the image; do not install at boot |
| **CDN cache hit ratio** | Determines whether your egress bill is large or enormous |

> [!TIP]
> **The trade to understand is encode time against delivery bandwidth.** A slower preset or a modern codec produces a smaller file — you pay once in compute and save on every subsequent view. For a video watched a million times that is obviously correct; for one watched twice it is obviously not. Encode popular content harder, and let view count decide.

---

## 15. Security Considerations

> [!CAUTION]
> **Uploaded media is executable input to a decoder, and this pipeline processes it at scale on behalf of strangers.** The two dominant risks are decoder exploitation and resource exhaustion — a small crafted file that expands into an enormous decode, or a stream designed to loop forever. Both are handled by limits and isolation, not by validation alone.

- **Isolate the transcoder** — a container with no credentials beyond its job, no unnecessary network, read-only root, and a hard wall-clock limit; a microVM if the content is fully untrusted
- **Never trust the file extension or the client-supplied MIME type** — probe the container
- **Hard limits on duration, resolution, dimensions and output size** before work begins
- **Presigned URLs scoped to one key, expiring in minutes** — for upload and for playback
- **Private buckets with a CDN and origin access control**; segments served through signed URLs if the content is restricted
- **Strip metadata** — uploaded media frequently contains GPS coordinates and device identifiers
- **Rate limit uploads per account**, since transcoding is an expensive thing to let strangers trigger
- **Cost alarms**, because an abuse campaign here is a financial attack

---

## 16. Mental Model

> [!NOTE]
> **This is a dry cleaner, not a tailor.**
>
> You hand over the coat and receive a ticket immediately — nobody stands at the counter while the work happens. The garments go onto a rail, the machines run in the back at whatever rate the day requires, and staff are called in when the rail is full and sent home when it is empty. A single item that comes out wrong is redone, not the whole rail. And when you collect, the coat comes from the front rack nearest the door — never from the machine room.

---

## 17. Complete request flow

```text
User selects a 4 GB video
    ↓
Client requests an upload; the API checks auth, quota and plan limits
    ↓
Returns a presigned MULTIPART upload; a job row is created: status = uploading
    ↓
Browser uploads 400 parts in parallel, directly to S3
    ↓
The connection drops at part 250 → only that part is retried
    ↓
Upload completed; S3 emits ObjectCreated
    ↓
─────────────── validation ───────────────
Orchestrator probes the file: real container, 118 minutes, 3840×2160, H.264
    ↓
Within plan limits; malware scan clean → status = validating → transcoding
    ↓
─────────────── planning ───────────────
Ladder chosen from the source: 2160p, 1080p, 720p, 480p, 360p, audio-only
    ↓
Split at keyframes into 240 segments × 6 renditions = 1,440 jobs, queued
    ↓
─────────────── the work ───────────────
Queue depth rises → the worker fleet scales from 0 to 120 SPOT instances
    ↓
Each worker: claim a message, transcode one segment with FFmpeg in an isolated
container, write to a DETERMINISTIC key, acknowledge
    ↓
Visibility timeout is 30 minutes; the longest segment takes 4 → no double work
    ↓
Progress written per segment → the UI shows a real percentage
    ↓
A spot instance receives a 2-minute termination notice
    → stops claiming new work; its in-flight message is not acknowledged
    → the queue redelivers it to another worker
    → deterministic keys mean even a duplicate write is harmless
    ↓
One segment fails three times (a corrupt region in the source)
    → sent to the dead letter queue → alarm → a human looks at it
    → the other 1,439 segments completed
    ↓
─────────────── packaging ───────────────
Segments concatenated per rendition; HLS and DASH manifests generated
    ↓
Thumbnails and a preview sprite extracted
    ↓
status = ready; the uploader is notified
    ↓
Lifecycle rule moves the original master to cold storage after 30 days
    ↓
─────────────── playback ───────────────
Viewer requests the manifest (short cache lifetime) via the CDN
    ↓
Player starts at a conservative bitrate → playback begins in under 2 seconds
    ↓
Bandwidth measured as good → steps up to 1080p
    ↓
Segments served from CDN cache (long lifetime, immutable) — the bucket is barely touched
    ↓
Bandwidth halves mid-stream → the player drops to 480p
    ↓
Quality degrades. Playback does not stop.
    ↓
─────────────── the cost review ───────────────
Cost per minute of video is tracked as a first-class metric
    ↓
Spot saves ~85% of compute; the CDN saves most of the egress
    ↓
Rarely-watched videos are encoded with a faster preset; popular ones re-encoded harder
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> Accept the upload directly to object storage with a presigned multipart URL, put the expensive work behind a queue with idempotent workers on spot capacity, keep authoritative job state in a database, and serve immutable outputs from a CDN.

---

## 19. Common Mistakes

- **Uploading through the application tier**, inheriting bandwidth, memory and timeout problems
- **Transcoding inside a web request**, or in a serverless function with an execution limit
- **A visibility timeout shorter than the longest job**, causing duplicate work
- **Non-deterministic output keys**, so a duplicate worker corrupts the result
- **No validation**, paying to transcode files that were never valid
- **Running FFmpeg on untrusted input without isolation or limits**
- **No dead letter queue**, so failures vanish and jobs sit "processing" forever
- **No progress reporting**, so users and operators cannot distinguish slow from stuck
- **Serving segments straight from object storage**, and paying full egress
- **On-demand instances** for a workload that is ideal for spot
- **No lifecycle rules**, so masters and abandoned uploads accumulate forever
- **Only object storage, no database row**, leaving orphaned files nobody can account for
- **Not stripping metadata**, publishing users' GPS coordinates
- **Building it at all** when a managed service would have been cheaper than the engineering

---

## 20. Open Source Technologies

- **FFmpeg** — the entire industry runs on it; learn `-c:v`, `-crf`, `-preset`, `-g` and `-hls_time`
- **ffprobe** — validation and inspection, the step that saves money
- **SVT-AV1**, **x264**, **x265**, **NVENC** — encoders, from quality-first to speed-first
- **Shaka Packager**, **Bento4** — HLS and DASH packaging
- **VMAF** — Netflix's perceptual quality metric; the honest way to compare encoder settings
- **Shaka Player**, **hls.js**, **dash.js**, **video.js** — adaptive bitrate players
- **tus** — an open resumable upload protocol, if presigned multipart does not fit
- **Temporal**, **Conductor**, **Argo Workflows** — orchestrate multi-stage jobs with retries
- **Celery**, **BullMQ**, **RabbitMQ**, **SQS** — the queue and worker layer
- **ClamAV** — malware scanning on uploads
- **MinIO** — S3-compatible storage for local development of this pipeline

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Find one place in your system where a file passes through your application. Replace it with a presigned URL.
- [ ] Compare your queue's visibility timeout with your longest job. Fix it if it is shorter.
- [ ] Check that running the same job twice produces the same output at the same key.
- [ ] Confirm every asynchronous job has a dead letter queue with an alarm.
- [ ] Identify one workload that could run on spot capacity and estimate the saving.
- [ ] Work out your cost per unit of work — per minute of video, per document, per image. If you do not know it, that is the first task.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
presigned multipart upload → S3 → event → orchestrator → queue
    → idempotent spot workers (FFmpeg, isolated) → immutable segments → CDN
    → job state in a database throughout
```

## 2. Request Flow

```text
Input       a very large file from an unreliable connection
    ↓
Processing  validated cheaply, split into parallel idempotent jobs on interruptible capacity
    ↓
Output      immutable renditions served from a CDN, and a user who waited seconds
```

## 3. Real-World Usage

Every platform accepting user media converges on this pipeline, and the reason it is worth studying is that it generalises so cleanly: replace "transcode" with "convert", "transcribe", "resize" or "run inference" and the architecture is unchanged. The details that separate a working pipeline from a painful one are unglamorous — visibility timeouts, deterministic keys, dead letter queues, and a CDN in front of the outputs.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | An asynchronous pipeline that turns one uploaded master into many deliverable renditions |
| **Why is it built this way?** | Because the work is far too slow and expensive to happen inside a request |
| **Where does it belong?** | Between object storage and a CDN, with a queue in the middle |
| **What should I take from it?** | Presigned uploads, idempotent queued workers on spot, immutable outputs, CDN delivery |
