# Docker

> **In one line —** packaging an application with everything it needs into an image that runs identically anywhere, using kernel features that were already there and an interface that finally made them usable.

| | |
|---|---|
| **Category** | Containerisation |
| **Architectural Layer** | Infrastructure |
| **Related notes** | [Docker Compose](Docker%20Compose.md) · [Kubernetes](Kubernetes.md) · [CI CD](CI%20CD.md) · [Process](../02%20-%20Computer%20Science%20Fundamentals/Process.md) · [Linux](../02%20-%20Computer%20Science%20Fundamentals/Linux.md) · [EC2](../12%20-%20Cloud%20Architecture/EC2.md) |

---

## 1. Short Definition

*What is it?*

Docker builds and runs **containers**: isolated processes that see their own filesystem, network and process tree, while sharing the host's kernel. An **image** is the immutable, layered filesystem; a **container** is one running instance of it.

---

## 2. Problem

*What engineering problem does it solve?*

```text
"It works on my machine"
    ↓
Different OS, different library versions, different Python patch level
Different environment variables, a package installed two years ago
    ↓
Deployment = a wiki page of steps, performed by hand, slightly differently each time
    ↓
And a virtual machine solves it at the cost of a whole guest OS per application
```

> [!IMPORTANT]
> **Docker's contribution was not the isolation technology — Linux namespaces and cgroups already existed, and so did LXC. It was the image format and the developer experience.** A layered, cacheable, shareable filesystem plus a one-file build definition plus a registry made containers something an ordinary developer would actually use. That packaging idea is what changed the industry, and it is why the artefact — not the runtime — is the part that matters most.

---

## 3. Architecture Position

```text
    Dockerfile ──build──► IMAGE ──push──► REGISTRY ──pull──► CONTAINER
                          (layers,        (GHCR, ECR,        (a process,
                           immutable,      Docker Hub)        isolated)
                           tagged)
                                                                  │
    ┌─────────────────────────────────────────────────────────────┘
    ▼
  HOST KERNEL  ← shared; this is why containers are lightweight
    namespaces (what the process can SEE)
    cgroups    (how much it can USE)
    union filesystem (layers)
```

---

## 4. Containers versus virtual machines

```text
VIRTUAL MACHINE                    CONTAINER
its own kernel and full OS         shares the host kernel
gigabytes                          megabytes
boots in tens of seconds           starts in milliseconds
hardware-level isolation           kernel-level isolation
runs any OS                        must match the host kernel (Linux on Linux)
```

> [!CAUTION]
> **A container is not a security boundary of the same strength as a VM.** Everything shares one kernel, so a kernel vulnerability or a misconfigured container can cross the line. For your own services this is fine. For running genuinely untrusted code — customer-supplied builds, arbitrary user submissions — use a VM, a microVM like Firecracker, or gVisor. Treating containers as equivalent to VMs for multi-tenant untrusted workloads is a recurring and serious mistake.

---

## 5. Layers, and why the Dockerfile order matters

```text
Every instruction creates a LAYER, cached by content.
A changed layer invalidates every layer after it.

BAD                                  GOOD
COPY . .                             COPY package*.json ./
RUN npm ci                           RUN npm ci
                                     COPY . .
    ↓                                    ↓
any source change reinstalls         source changes reuse the
all dependencies (90 s)              dependency layer (2 s)
```

> [!TIP]
> **Order instructions from least to most frequently changed. This single habit is the difference between a 2-second and a 2-minute rebuild**, on every build, for the life of the project. Dependencies change weekly; your source changes hourly — so dependencies must come first.

---

## 6. Multi-stage builds

```dockerfile
# ---- build stage: has the compiler, the SDK, the dev dependencies
FROM node:22 AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# ---- runtime stage: has only what is needed to RUN
FROM node:22-slim
WORKDIR /app
ENV NODE_ENV=production
COPY package*.json ./
RUN npm ci --omit=dev && npm cache clean --force
COPY --from=build /app/dist ./dist
USER node
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

> [!IMPORTANT]
> **Multi-stage builds are not an optimisation; they are a security control.** Build tools, compilers, source code, `.git` and credentials used during the build stay behind in a stage that is never shipped. A single-stage image typically carries a compiler and package manager into production — every one of which is a tool an attacker can use after landing in your container.

---

## 7. Images: keep them small and boring

```text
node:22            ~1.1 GB   full Debian, compilers, everything
node:22-slim       ~200 MB   the sensible default
node:22-alpine     ~130 MB   musl libc — watch for native module issues
distroless         ~80 MB    no shell, no package manager — hardest to attack
scratch            ~5 MB     static binaries only (Go, Rust)
```

```text
Smaller image →  faster pulls, faster scaling, fewer CVEs to triage
```

> [!TIP]
> **Prefer `-slim` by default and distroless where you can.** Alpine looks appealing but musl libc causes real, hard-to-diagnose problems with native modules and DNS behaviour in some ecosystems — the saving is rarely worth an afternoon of debugging. Distroless is the genuinely strong option: with no shell in the image, the standard post-exploitation toolkit simply is not there.

---

## 8. Containers are stateless and ephemeral

```text
WRITE INSIDE A CONTAINER   → gone when it is replaced
    ↓
STATE GOES ELSEWHERE
    a volume            for databases you run yourself
    a managed database  for real data (see RDS)
    object storage      for files (see S3)
    a cache             for sessions

CONFIGURATION comes from ENVIRONMENT VARIABLES, not from files baked in
    → one image, promoted unchanged through every environment
```

> [!IMPORTANT]
> **One image, many environments. If you build a separate image per environment, you have lost the guarantee you came for**, because staging is no longer running the bytes production will run. Configuration is injected at run time; the artefact is identical everywhere and tagged with a commit SHA.

---

## 9. Real World Example

- **Every CI pipeline** — a job runs in a container so the build environment is defined in code.
- **Local development** — the whole stack, including the database, started with one command; see [Docker Compose](Docker%20Compose.md).
- **The unit of deployment on Kubernetes and ECS** — the scheduler's job is to place containers; see [Kubernetes](Kubernetes.md).
- **Model serving** — CUDA and framework versions pinned inside the image; see [Model Serving](../11%20-%20AI%20Engineering/Model%20Serving.md).
- **Reproducible builds** — a compiler toolchain from 2019, still working, because it is in an image.
- **Legacy applications**, containerised without rewriting, to escape a dying host OS.

---

## 10. Communication and Dependencies

- **A Linux kernel** — on macOS and Windows a lightweight VM provides it
- **A registry** — GHCR, ECR, Docker Hub; images must live somewhere
- **A `.dockerignore` file** — without it, `.git`, `node_modules` and secrets go into the build context
- **An orchestrator for production** — plain `docker run` on a server has no scheduling, health management or rollout
- **A vulnerability scanner** in CI; see [CI CD](CI%20CD.md)
- **A health check**, so an orchestrator can tell running from working

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Use Docker for essentially any server-side application, for local development environments, and for CI jobs. It is the standard artefact format, and building one is the price of entry to every modern deployment platform.

> [!CAUTION]
> - **Not for running untrusted third-party code** in a multi-tenant way — use a VM or microVM
> - **Not as a production orchestrator on its own** — `docker run` with a restart policy is not high availability
> - **Not for stateful databases in production** unless you have deliberately designed the storage; a managed database is usually better
> - **Not for desktop GUI applications**, where the effort exceeds the benefit
> - **Not to hide an unmaintained base image** — a container does not patch anything
> - **Not for a single static binary** deployed to one machine; you may not need it

---

## 12. Advantages and Disadvantages

**Advantages**
- The same artefact runs identically on a laptop, in CI and in production
- Starts in milliseconds, with negligible runtime overhead
- Layered caching makes builds fast and pushes small
- Dependencies are explicit and versioned in a file
- The universal input to every orchestrator and cloud platform
- Local environments become disposable and reproducible
- Immutable artefacts make rollback trivial

**Disadvantages**
- **Weaker isolation than a VM** — one shared kernel
- Persistent state requires deliberate design
- Easy to build large, insecure images by accident
- Networking has real subtleties, especially across platforms
- Another layer to debug when something is slow
- Registry storage and pull bandwidth cost money
- Every image is a dependency inventory you now own and must patch
- Docker Desktop is licensed for larger companies

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **CPU and memory** | Effectively native — it is just a process |
| **Startup** | Milliseconds, versus tens of seconds for a VM |
| **Network** | Bridge networking adds a small overhead; host networking removes it |
| **Storage (overlay)** | Fine for reads; write-heavy workloads want a volume |
| **Bind mounts on macOS/Windows** | **Significantly slow** — a common cause of sluggish local development |
| **Image size** | Directly affects pull time, which affects how fast you can scale |
| **Build cache** | The single biggest lever on developer and CI feedback time |

> [!TIP]
> **If local development feels slow on macOS, it is almost always the bind mount.** File system events and I/O across the VM boundary are expensive. Use named volumes for dependency directories, enable VirtioFS, or keep the heaviest workload out of the mount — and do not conclude that containers are slow, because on Linux the same setup is native speed.

---

## 14. Security Considerations

> [!CAUTION]
> **Containers run as root by default, and that root is the host's root if it ever escapes.** Add a non-root `USER`, drop capabilities, and set `read_only: true` with explicit writable mounts. Running as root inside a container is not required by anything — it is just the default nobody changed, and it converts a minor application bug into a much more serious problem.

- **`USER` a non-root account**, and `--cap-drop=ALL` with only what is needed added back
- **Never bake secrets into an image** — every layer is readable, and `docker history` shows the build steps; use build secrets or runtime injection
- **A `.dockerignore` file** so `.git`, `.env` and credentials never enter the build context
- **Pin base images by digest**, not by a moving tag like `latest`
- **Scan images in CI** with Trivy or Grype, and fail on critical findings
- **Rebuild regularly** — an image built six months ago has six months of unpatched CVEs
- **Never mount the Docker socket** into a container you do not fully trust; it is equivalent to root on the host
- **No `--privileged`** unless you can explain precisely why
- **Sign and verify images** with cosign so only what your pipeline built can run

---

## 15. Mental Model

> [!NOTE]
> **An image is a shipping container; the host is the ship.**
>
> The container is sealed at the factory with everything inside, and every crane, truck and ship handles it identically because the outside is standardised — that interchangeability, not the box, is the innovation. Many containers share one ship's engine, which is why they are cheap compared with sending a separate vessel each time. And the walls are steel, not a vault: good enough for your own cargo, not what you would choose for storing something that actively wants to get out.

---

## 16. Mini Architecture Diagram

```text
   Dockerfile                    .dockerignore excludes .git, .env
       │
   ┌───▼──────────── BUILD (multi-stage) ─────────────┐
   │  stage 1: compiler, SDK, dev deps, source        │
   │           └── discarded, never shipped           │
   │  stage 2: runtime only + built output            │
   │           USER node · no shell if distroless     │
   └───┬──────────────────────────────────────────────┘
       │ tag with the commit SHA, sign with cosign
       ▼
   REGISTRY (GHCR / ECR)  ── scanned by Trivy in CI
       │
       │ pull
       ▼
   ┌─────────── HOST ────────────────────────────────┐
   │  container A      container B      container C   │
   │  non-root         non-root         non-root      │
   │  read-only fs     mem/cpu limits   health check  │
   │  ─────────────────────────────────────────────   │
   │  SHARED KERNEL: namespaces + cgroups             │
   └──────────────────────────────────────────────────┘
       │                    │
   env vars for config   volumes / managed DB / S3 for state
```

---

## 17. Complete Request Flow

```text
Developer edits one source file
    ↓
docker build: layers for the base image and npm ci are CACHE HITS
    ↓
Only the COPY of source and the build step re-run — 4 seconds
    ↓
Multi-stage: the compiler and dev dependencies stay in stage 1
    ↓
Final image is 180 MB, runs as a non-root user, has no package manager
    ↓
CI scans it — one high CVE in a transitive dependency → build fails
    ↓
Dependency bumped, rebuilt, scan clean
    ↓
Pushed to GHCR as app:<commit-sha> and signed
    ↓
─────────────── deployment ───────────────
The orchestrator pulls the image (fast, because it is small)
    ↓
Starts the container with env vars for THIS environment — the image is unchanged
    ↓
CPU and memory limits applied via cgroups
    ↓
Health check passes → traffic routed to it
    ↓
State goes to the managed database and object storage; the container writes nothing durable
    ↓
─────────────── an incident ───────────────
A bug is found; the previous image tag still exists
    ↓
Roll back by starting the old image — seconds, and identical bytes to before
    ↓
─────────────── an attacker ───────────────
An application vulnerability yields code execution inside the container
    ↓
No shell (distroless), non-root user, read-only filesystem, no Docker socket
    ↓
Capabilities dropped, so no network reconfiguration or privilege escalation
    ↓
Container replaced from the image; nothing persisted
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> Order Dockerfile layers from least to most changeable, use multi-stage builds so no compiler reaches production, run as a non-root user with a small base image, keep state and configuration outside the container, and promote one identical image through every environment.

---

## 19. Common Mistakes

- **`COPY . .` before installing dependencies**, destroying the build cache
- **Single-stage builds** that ship compilers, source and build tooling
- **Running as root** because it is the default
- **Secrets baked into layers**, visible in `docker history` forever
- **No `.dockerignore`**, so `.git` and `.env` enter the build context
- **`FROM node:latest`** — an unpinned, moving base image
- **Never rebuilding**, so the image accumulates unpatched vulnerabilities
- **Writing state inside the container**, then losing it on replacement
- **A different image per environment**, discarding the whole guarantee
- **Mounting the Docker socket** into an application container
- **`--privileged`** as a workaround for a permissions problem
- **Alpine chosen reflexively**, then hours lost to musl and native modules
- **No health check**, so the orchestrator cannot distinguish running from working
- **Assuming container isolation equals VM isolation** for untrusted code

---

## 20. Open Source Technologies

- **Docker Engine**, **containerd**, **runc** — the runtime stack beneath it all
- **Podman** — daemonless and rootless by default; a drop-in for most uses
- **BuildKit / buildx** — faster builds, build secrets, multi-platform images
- **Kaniko**, **Buildah** — build images without a privileged daemon, useful in CI
- **Trivy**, **Grype**, **Dockle** — scan images and lint their configuration
- **hadolint** — lints Dockerfiles and catches most of section 19
- **dive** — inspect layers and find what is bloating an image
- **distroless**, **chainguard images** — minimal, low-CVE base images
- **cosign / Sigstore** — sign and verify images
- **gVisor**, **Firecracker**, **Kata** — stronger isolation when you need it

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Run `dive` on your largest image and find the three biggest layers.
- [ ] Check whether your production containers run as root. Fix one that does.
- [ ] Reorder a Dockerfile so dependencies are cached, and measure the rebuild time.
- [ ] Convert a single-stage build to multi-stage and compare image sizes.
- [ ] Run `hadolint` and `trivy` against one image and read the findings.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Dockerfile → layered image (multi-stage) → registry → container on a shared kernel
                                                     └── config from env, state outside
```

## 2. Request Flow

```text
Input       a Dockerfile and a build context, with dependencies declared first
    ↓
Processing  cached layers, a discarded build stage, a minimal signed runtime image
    ↓
Output      one immutable artefact that runs identically everywhere
```

## 3. Real-World Usage

Docker's lasting win is the **artefact**, not the daemon. Kubernetes, ECS, Cloud Run and most CI systems consume OCI images; several of them no longer use Docker itself to run them. What every team standardised on is the idea that a deployable unit is an immutable, layered, signed image tagged with the commit that produced it.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A tool for building layered images and running them as isolated processes |
| **Why does it exist?** | Because environment differences made deployment unreliable, and VMs were too heavy |
| **Where does it belong?** | As the unit of build, test and deployment for nearly all server software |
| **When should I use it?** | Almost always — but not as a security boundary for untrusted code |
