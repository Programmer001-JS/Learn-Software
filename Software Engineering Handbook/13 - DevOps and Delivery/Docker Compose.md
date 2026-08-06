# Docker Compose

> **In one line —** one YAML file that starts your whole stack on a laptop with a single command; the best onboarding tool in software, and a poor production platform.

| | |
|---|---|
| **Category** | Multi-container Orchestration *(local)* |
| **Architectural Layer** | Development |
| **Related notes** | [Docker](Docker.md) · [Kubernetes](Kubernetes.md) · [CI CD](CI%20CD.md) · [PostgreSQL](../08%20-%20Databases%20and%20Data/PostgreSQL.md) · [Redis](../08%20-%20Databases%20and%20Data/Redis.md) |

---

## 1. Short Definition

*What is it?*

Compose declares a set of containers, their networks and their volumes in a single `compose.yaml`, then starts them together with `docker compose up`. It turns "install and configure eight things" into one command.

---

## 2. Problem

*What engineering problem does it solve?*

```text
A new engineer joins on Monday
    ↓
README: install PostgreSQL 16, Redis, MinIO, create a database,
run migrations, set 14 environment variables, start three services
    ↓
Two days lost, and their versions differ subtly from everyone else's
    ↓
Six months later nobody remembers why one service needs a specific flag
```

> [!IMPORTANT]
> **Compose's real value is that the development environment becomes reviewable code.** When someone adds a dependency, the pull request shows it. When a version changes, everyone gets it on their next `up`. The alternative — a README that drifts from reality — is how "works on my machine" survives even after everything is containerised.

---

## 3. Architecture Position

```text
    compose.yaml
        │
   docker compose up
        │
   ┌────▼──────── a private network ("compose_default") ──────┐
   │                                                           │
   │   api  ──────► postgres        service names resolve      │
   │    │    ──────► redis          via DNS: "postgres:5432"   │
   │    │    ──────► minio                                     │
   │   worker ─────► redis                                     │
   │                                                           │
   │   named volumes: pgdata, miniodata  (survive `down`)      │
   └───────────────────────────────────────────────────────────┘
        │
   only the ports you publish are reachable from the host
```

> [!TIP]
> **Containers reach each other by service name, on the container's own port.** From `api`, the database is `postgres:5432` — not `localhost:5432`, and not the host-published port. Getting this wrong is the single most common Compose confusion, and it also explains why you should publish as few ports as possible.

---

## 4. A realistic file

```yaml
services:
  api:
    build:
      context: .
      target: dev                  # a dev stage of your multi-stage Dockerfile
    ports: ["3000:3000"]
    environment:
      DATABASE_URL: postgres://app:app@postgres:5432/app
      REDIS_URL: redis://redis:6379
    volumes:
      - .:/app                     # live reload
      - /app/node_modules          # keep the container's deps, not the host's
    depends_on:
      postgres:
        condition: service_healthy # WAIT for readiness, not just for start
      redis:
        condition: service_started

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app       # a local-only value; never a real secret
      POSTGRES_DB: app
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      retries: 10

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]

volumes:
  pgdata:
```

---

## 5. `depends_on` does not mean "ready"

```text
WITHOUT a health check
    depends_on: [postgres]
        ↓
    Compose starts postgres, then IMMEDIATELY starts api
        ↓
    postgres needs 3 seconds to accept connections
        ↓
    api crashes on startup — every single time, on a fast machine
```

```text
WITH a health check
    depends_on:
      postgres:
        condition: service_healthy
        ↓
    api starts only once pg_isready succeeds
```

> [!CAUTION]
> **Start order is not readiness order, and this causes the majority of "Compose is flaky" complaints.** Add a health check to every stateful service and use `condition: service_healthy`. Even then, your application should retry its initial connection — because in production nothing guarantees the database is up when your process starts either.

---

## 6. Volumes: the two kinds, and when each hurts

```text
NAMED VOLUME        volumes: [pgdata:/var/lib/postgresql/data]
    managed by Docker, fast on every platform
    survives `down`, removed by `down -v`
    → use for databases and any real data

BIND MOUNT          volumes: ['.:/app']
    the host directory appears inside the container
    → use for source code and live reload
    ⚠ SLOW on macOS and Windows
```

> [!TIP]
> **The `- /app/node_modules` anonymous volume trick matters more than it looks.** A bind mount of `.` would otherwise hide the container's installed dependencies behind the host's — which were compiled for a different platform. Masking that one path with a container-owned volume is the standard fix, and skipping it produces native-module errors that make no sense.

---

## 7. Compose in CI

```text
docker compose up -d --wait     # exits when everything is HEALTHY
docker compose exec -T api npm test
docker compose down -v

WHY IT IS GOOD HERE
    integration tests against a real database, not a mock
    the same definition developers use locally
    torn down completely afterwards
```

> [!TIP]
> **`--wait` is the flag that makes Compose usable in CI**, because it blocks until health checks pass instead of returning as soon as containers exist. Without it, pipelines end up with a `sleep 15` that is simultaneously too slow on a good day and too short on a bad one.

---

## 8. Why not in production

```text
COMPOSE ON ONE SERVER              WHAT PRODUCTION NEEDS
one host                            multiple hosts, survive one dying
restart: always                     rescheduling onto healthy capacity
manual `up -d` to deploy            rolling updates with health gating
no rollout control                  canary, rollback, progressive delivery
no secret management                real secret injection
scale = edit and re-run             autoscaling on demand
```

> [!IMPORTANT]
> **Compose on a single server is a legitimate choice for small, internal, non-critical workloads — but be honest that it is a single server.** No amount of `restart: always` survives the host failing, and there is no rolling update, so every deployment is a brief outage. For a side project or an internal tool, that may be entirely acceptable. For anything with an availability expectation, the honest options are a managed platform, ECS, or Kubernetes. Docker Swarm exists and works, but its ecosystem has effectively stopped moving.

---

## 9. Real World Example

- **Local development for any multi-service application** — the dominant use, by a wide margin.
- **Integration tests in CI** against real PostgreSQL, Redis and an S3-compatible store.
- **Demo and evaluation environments** — one command for a reviewer or a customer.
- **Self-hosted tools** on a small VPS — most open-source projects ship a Compose file for exactly this.
- **Reproducing a bug** with an exact dependency version combination.
- **Teaching and workshops**, where setup time would otherwise consume the session.

---

## 10. Communication and Dependencies

- **Docker Engine** — Compose is a thin orchestration layer over it; see [Docker](Docker.md)
- **`.env` for local values**, and `.env` in `.gitignore`
- **Health checks** on every stateful service
- **A migration step** — run migrations explicitly, not implicitly on start
- **`compose.override.yaml`** for per-developer differences without editing the shared file
- **Profiles** to keep optional services out of the default `up`

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Use Compose for local development and for integration tests in CI. Every repository with more than one moving part should have a `compose.yaml` that works from a fresh clone with one command.

> [!CAUTION]
> - **Not for production with availability requirements** — it is one host with no rescheduling
> - **Not as a substitute for learning your orchestrator** — a Compose file is not a Kubernetes manifest, and `kompose` output is a starting point at best
> - **Not for secrets** — values in the file and in `.env` are plaintext
> - **Not for large stacks on a laptop** — fifteen services will exhaust memory and patience; use profiles
> - **Not with `latest` tags**, which quietly destroys the reproducibility you came for

---

## 12. Advantages and Disadvantages

**Advantages**
- One command from a fresh clone to a working stack
- The environment is versioned and reviewed with the code
- Real dependencies in tests instead of mocks
- Trivial to tear down and start clean
- Service discovery by name, with no configuration
- Excellent for demos, workshops and self-hosting

**Disadvantages**
- **Single host** — no rescheduling, no real high availability
- No rolling updates or rollout control
- Secrets are plaintext
- Bind mounts are slow on macOS and Windows
- Divergence from production, which can hide real problems
- Large stacks are heavy on a developer machine
- Effectively frozen in feature terms compared with orchestrators

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Container overhead** | Negligible — as with any container |
| **Bind mounts on macOS/Windows** | The main cause of slow local development |
| **Many services** | Memory adds up quickly; use `profiles` to start subsets |
| **Startup time** | Dominated by health check intervals; tune them down for local use |
| **Image pulls** | First `up` is slow; cached afterwards |
| **CI usage** | `--wait` plus tuned health checks keeps pipelines predictable |

> [!TIP]
> **If local development is slow, look at bind mounts and health check intervals before anything else.** A 30-second health check interval means a 30-second wait for no reason, and a bind-mounted `node_modules` on macOS can be an order of magnitude slower than a named volume.

---

## 14. Security Considerations

> [!CAUTION]
> **Compose files invite committed secrets, and committing one is permanent.** Real API keys and passwords end up in `compose.yaml` because it is convenient, then into Git history, then into every clone. Keep only obviously-fake local values inline, put anything else in a `.env` that is gitignored, and commit a `.env.example` listing the names.

- **Never publish a database port to `0.0.0.0`** on a shared or cloud host — Compose port publishing bypasses the host firewall on some setups
- **Do not mount the Docker socket** into a service unless you fully trust it; it is root on the host
- **Pin image tags** — `postgres:16-alpine`, not `postgres:latest`
- **Non-root users** in your own images, exactly as in production; see [Docker](Docker.md)
- **`docker compose down -v` removes volumes** — easy to run by accident against something you cared about
- **If you do run it on a server**: a firewall, TLS termination in front, pinned versions, and an actual backup of the volumes

---

## 15. Mental Model

> [!NOTE]
> **Compose is a stage manager for a rehearsal, not a touring production.**
>
> It gets every actor and prop into place with one call, in the right order, on one stage — which is exactly what you want when rehearsing. It has no plan for the theatre burning down, no understudies, and no way to swap an actor mid-performance without stopping the show. That is fine in a rehearsal room. It is why the real tour hires a different kind of crew.

---

## 16. Mini Architecture Diagram

```text
   git clone && docker compose up --wait
        │
   ┌────▼───────────────── compose network ─────────────────────┐
   │                                                             │
   │   ┌─────────┐ published 3000        ┌──────────────┐        │
   │   │   api   │◄──── host:3000        │   worker     │        │
   │   │ bind mnt│                       │  (no ports)  │        │
   │   │ /app    │                       └──────┬───────┘        │
   │   └──┬───┬──┘                              │                │
   │      │   │        ┌────────────────────────┘                │
   │      │   └───────►│ redis:6379   healthcheck: PING          │
   │      │            └─────────────────────────────────────────│
   │      │            ┌─────────────────────────────────────────│
   │      └───────────►│ postgres:5432  healthcheck: pg_isready  │
   │                   │   volume: pgdata (survives `down`)      │
   │                   └─────────────────────────────────────────│
   └─────────────────────────────────────────────────────────────┘

   depends_on: condition: service_healthy   ← the line that prevents flakiness
```

---

## 17. Complete Request Flow

```text
New engineer: git clone, then docker compose up --wait
    ↓
Images pulled; the api image is built from the dev stage of the Dockerfile
    ↓
postgres and redis start; health checks begin polling
    ↓
pg_isready succeeds after 3 s → postgres marked healthy
    ↓
ONLY NOW does api start — because of condition: service_healthy
    ↓
api connects to postgres:5432 by service name over the private network
    ↓
Migrations run as an explicit step, not implicitly on boot
    ↓
Source is bind-mounted, so saving a file reloads inside the container
    ↓
node_modules is masked by an anonymous volume — the container's own, correct build
    ↓
Working stack in 90 seconds, on the first day
    ↓
─────────────── in CI ───────────────
docker compose up -d --wait → exits only when everything is healthy
    ↓
Integration tests run against real PostgreSQL and Redis
    ↓
docker compose down -v → nothing left behind
    ↓
─────────────── someone adds a dependency ───────────────
A pull request adds a `minio` service and a new environment variable
    ↓
Reviewed like any other change; every developer gets it on their next `up`
    ↓
No README to update, no message on Slack saying "you need to install…"
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> Compose is the best onboarding and integration-testing tool available — health checks with `service_healthy`, named volumes for data, pinned image tags, no secrets in the file — and it is a single host, so treat production use as a deliberate trade rather than an oversight.

---

## 19. Common Mistakes

- **`depends_on` without health checks**, producing startup races that look like flakiness
- **Connecting to `localhost` instead of the service name** from inside a container
- **Secrets committed** in `compose.yaml` or a tracked `.env`
- **`latest` tags**, discarding reproducibility
- **A bind-mounted `node_modules`**, causing native module failures
- **No `.env.example`**, so nobody knows which variables exist
- **Running it in production** while expecting orchestrator behaviour
- **Publishing database ports** on a cloud host, exposing them to the internet
- **`down -v` run casually** against data someone needed
- **Fifteen services in the default `up`** instead of using profiles
- **Migrations run implicitly on container start**, racing with each other when scaled

---

## 20. Open Source Technologies

- **Docker Compose** (v2, the `docker compose` plugin) — the file format is now the Compose Specification
- **Podman Compose** — the same file, daemonless
- **Testcontainers** — programmatic containers from inside your test suite; often better than Compose for integration tests
- **MinIO** — an S3-compatible store for local development; see [S3](../12%20-%20Cloud%20Architecture/S3.md)
- **LocalStack** — emulate AWS services locally
- **Mailpit**, **MailHog** — catch outbound email in development
- **Tilt**, **Skaffold**, **DevSpace** — the equivalent inner loop against Kubernetes
- **direnv**, **dotenv-linter** — manage and validate local environment files

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Clone your own project into a fresh directory and time `docker compose up` to a working stack.
- [ ] Add a health check to every stateful service and switch to `condition: service_healthy`.
- [ ] Check whether any real credential appears in your Compose file or a tracked `.env`.
- [ ] Add `--wait` to your CI pipeline and delete the `sleep`.
- [ ] Move your integration tests onto the same Compose definition developers use.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
compose.yaml → containers on one private network → named volumes for data
                        └── health checks gate startup order
```

## 2. Request Flow

```text
Input       one declarative file describing services, networks and volumes
    ↓
Processing  images pulled or built, containers started in health-gated order
    ↓
Output      a complete working stack on one host, from one command
```

## 3. Real-World Usage

Compose settled into two roles it does better than anything else: the local development environment and the integration test harness. Almost every open-source project ships a Compose file for evaluation, and almost every team uses one to onboard. Its production role shrank as orchestrators matured — which is fine, because being the best rehearsal room is a genuinely valuable job.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A declarative file that runs a multi-container stack on one host |
| **Why does it exist?** | Because setting up a development environment by hand never stays correct |
| **Where does it belong?** | On developer machines and in CI |
| **When should I use it?** | Always locally; in production only as a conscious single-host trade-off |
