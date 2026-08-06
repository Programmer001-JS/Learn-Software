# Kubernetes

> **In one line —** a control loop that continuously moves your cluster towards a declared desired state; enormously capable, and it will happily become your team's main project if you let it.

| | |
|---|---|
| **Category** | Container Orchestration |
| **Architectural Layer** | Infrastructure |
| **Related notes** | [Docker](Docker.md) · [Docker Compose](Docker%20Compose.md) · [CI CD](CI%20CD.md) · [Deployment Strategies](Deployment%20Strategies.md) · [Horizontal Scaling](../14%20-%20Scalability%20and%20Reliability/Horizontal%20Scaling.md) · [High Availability](../14%20-%20Scalability%20and%20Reliability/High%20Availability.md) |

---

## 1. Short Definition

*What is it?*

Kubernetes runs containers across a fleet of machines. You **declare** what should exist — this many replicas of this image, reachable at this address — and controllers work continuously to make reality match, including after failures.

---

## 2. Problem

*What engineering problem does it solve?*

```text
You have 40 containers and 6 machines
    ↓
Which container runs where? What happens when a machine dies?
How does traffic find a container whose IP changed?
How do you deploy a new version without downtime?
How do you scale one service and not the others?
    ↓
Doing this by hand is a full-time job, done inconsistently
```

> [!IMPORTANT]
> **Kubernetes' idea is reconciliation, not scheduling.** You never tell it to start a container; you declare that three should exist, and a controller notices whenever that is untrue and acts. Self-healing, rolling updates, autoscaling and rescheduling after a node failure are all the same mechanism applied to different resources. Once you see that loop, the API stops looking like a hundred unrelated features.

---

## 3. Architecture Position

```text
   CONTROL PLANE (managed for you on EKS/GKE/AKS)
     api-server  ─── the only way in; everything talks to it
     etcd        ─── the desired state, stored
     scheduler   ─── decides which node a new pod goes on
     controllers ─── the reconciliation loops
          │
   ───────┼─────────────────────────────────────────
          ▼
   NODES (your EC2 instances or equivalent)
     kubelet     ─── starts containers, reports health
     kube-proxy  ─── service networking
     container runtime (containerd)
          │
        PODS ─── one or more containers sharing a network namespace
```

---

## 4. The objects you actually need

```text
POD           the smallest unit — usually one container. You rarely create one directly.
DEPLOYMENT    manages a ReplicaSet of identical pods; handles rolling updates
SERVICE       a stable name and virtual IP in front of a changing set of pods
INGRESS       HTTP routing from outside the cluster (or a Gateway API resource)
CONFIGMAP     non-secret configuration
SECRET        base64-encoded, NOT encrypted by default  ⚠
NAMESPACE     a logical boundary for names, quotas and permissions
STATEFULSET   for stable identity and per-pod storage — databases, brokers
DAEMONSET     one pod per node — log shippers, agents
JOB / CRONJOB one-off and scheduled work
PVC           a claim on persistent storage
HPA           autoscaling on metrics
```

> [!TIP]
> **Deployment, Service, Ingress, ConfigMap and Secret cover the overwhelming majority of application workloads.** Learn those five properly and you can run real software. The rest of the API surface exists for specific needs and can wait until one of them is actually yours.

---

## 5. Requests and limits — the most consequential setting

```text
REQUESTS   what the scheduler reserves      → determines WHERE the pod lands
LIMITS     the hard ceiling                 → determines what happens under pressure

CPU over the limit      → THROTTLED (slow, but alive)
MEMORY over the limit   → OOMKilled (terminated immediately)
```

```text
NO REQUESTS SET
    the scheduler assumes ~nothing → packs the node → everything degrades together

LIMITS FAR ABOVE REQUESTS
    works fine until the node is busy, then unpredictable throttling
```

> [!CAUTION]
> **Most mysterious Kubernetes behaviour is a requests-and-limits problem.** A pod restarting under load is usually an unnoticed OOMKill, not an application bug. A service that is fast in staging and slow in production is often CPU throttling from a limit that is too low. Set memory request equal to limit for predictability, set CPU requests honestly, and be wary of tight CPU limits — throttling is measured in whole 100 ms periods and hurts latency more than people expect.

---

## 6. Probes: the difference between running and working

```text
STARTUP probe     "has it finished booting?"     → gives slow starters time
READINESS probe   "can it serve traffic NOW?"    → controls SERVICE membership
LIVENESS probe    "is it wedged? restart it"     → controls RESTARTS
```

```text
GET THESE WRONG AND
    no readiness probe   → traffic sent to a pod still starting up
    liveness = readiness → a dependency outage becomes a restart loop
    liveness too aggressive → pods killed during a GC pause or a slow start
```

> [!CAUTION]
> **Never make a liveness probe depend on a downstream service.** If your liveness check queries the database, a brief database problem causes Kubernetes to restart every pod simultaneously — turning a degradation into an outage, and preventing recovery. Liveness answers "is this process broken?" only. Readiness is where dependency awareness belongs, and even there be careful about removing all pods from a service at once.

---

## 7. Networking, in the order you meet it

```text
POD IP        every pod gets one; ephemeral and not to be relied on
SERVICE       ClusterIP: a stable virtual IP + DNS name inside the cluster
              NodePort / LoadBalancer: exposure outward
INGRESS       one load balancer, host- and path-based routing, TLS termination
              (Gateway API is the successor and is where the ecosystem is moving)
NETWORK POLICY  which pods may talk to which — DEFAULT IS EVERYTHING
```

> [!CAUTION]
> **By default, every pod in the cluster can reach every other pod, in every namespace.** Namespaces are a naming and permissions boundary, not a network boundary. A compromised frontend can talk directly to your database pod unless a NetworkPolicy forbids it. Adopt a default-deny policy per namespace and allow explicitly — this is the most common gap between a working cluster and a defensible one.

---

## 8. Secrets are not secret

```text
A Kubernetes Secret is BASE64, not encryption.
    → anyone with `get secrets` in the namespace can read it
    → stored in etcd, which must be separately encrypted at rest
    → visible in a pod's environment, and in `describe` output for some resources
```

```text
The realistic options
    external secret store (Vault, AWS Secrets Manager) + External Secrets Operator
    sealed-secrets / SOPS  → encrypted values safe to commit to Git
    cloud IAM identity for pods (IRSA on EKS, Workload Identity on GKE)
        → best of all: no secret exists to manage
```

> [!IMPORTANT]
> **Prefer identity over secrets wherever possible.** A pod that assumes a cloud role via IRSA needs no stored database password if the database supports IAM authentication, and no stored API key for cloud services at all. Every secret you eliminate is one you cannot leak, rotate or forget. See [IAM](../12%20-%20Cloud%20Architecture/IAM.md).

---

## 9. Do you need it?

```text
GOOD REASONS
    many services, many teams, deployed independently
    genuine need for bin-packing across a fleet
    complex rollout requirements — canary, progressive delivery at scale
    a platform team exists to own the cluster
    a portability requirement across clouds or on-premises

BAD REASONS
    three services and eight developers      → ECS/Fargate, App Runner, or a PaaS
    "it is what serious companies use"        → they also have platform teams
    a CV-driven decision                     → be honest about this one
```

> [!IMPORTANT]
> **Kubernetes has a large fixed cost that does not shrink with your workload.** Upgrades, CNI and CSI plugins, ingress controllers, RBAC, certificate rotation, node lifecycle and a dozen operators are ongoing work regardless of whether you run three services or three hundred. For a small team, ECS Fargate or a managed platform delivers most of the benefit for a fraction of the operational surface. Adopt Kubernetes when the complexity of your workload exceeds the complexity of the tool — not before.

---

## 10. Real World Example

- **Large multi-service platforms** with many teams deploying independently — the case it was built for.
- **Machine learning workloads** — GPU scheduling, queued training jobs, autoscaled inference; see [Model Serving](../11%20-%20AI%20Engineering/Model%20Serving.md).
- **GitOps delivery** — Argo CD reconciling the cluster to a Git repository, so the repo is the deployment record.
- **Multi-tenant SaaS**, using namespaces, quotas and network policies per tenant.
- **On-premises and regulated environments**, where a portable platform genuinely matters.
- **Batch processing** — Jobs and CronJobs replacing a fleet of cron servers.

---

## 11. Communication and Dependencies

- **Container images** in a registry; see [Docker](Docker.md)
- **A managed control plane** (EKS, GKE, AKS) unless you have a strong reason to run your own
- **An ingress controller or Gateway implementation** — nginx, Traefik, Envoy
- **A CNI plugin** that supports NetworkPolicy
- **A metrics pipeline** — metrics-server for HPA, Prometheus for everything else
- **A secrets solution**, external to Kubernetes
- **A delivery mechanism** — Helm or Kustomize, driven by Argo CD or Flux
- **A cluster autoscaler or Karpenter**, or scaling stops at your node count

---

## 12. When To Use / When NOT To Use

> [!TIP]
> Use Kubernetes when you have many independently deployed services, multiple teams, and someone whose job includes the platform. Use a **managed** control plane — running etcd yourself is a specialised skill with no product value.

> [!CAUTION]
> - **Not for a handful of services** — the operational cost dominates
> - **Not without someone owning it** — an unowned cluster decays into an unupgradeable liability
> - **Not for your primary database** unless you have a strong operational reason; a managed database is better; see [RDS](../12%20-%20Cloud%20Architecture/RDS.md)
> - **Not as a lift-and-shift target** for a monolith that would run fine on two instances
> - **Not self-managed control plane** on a small team
> - **Not as a portability guarantee** — cloud load balancers, storage classes and IAM integration are provider-specific anyway

---

## 13. Advantages and Disadvantages

**Advantages**
- Self-healing: failed pods and failed nodes are handled without a human
- Rolling updates and rollbacks as first-class operations
- Horizontal autoscaling of pods, and of nodes underneath them
- Efficient bin-packing across a fleet
- One declarative API for compute, networking, config and storage
- An enormous ecosystem, and skills that transfer between employers
- Extensible — CRDs and operators let you model your own resources

**Disadvantages**
- **A very large conceptual surface** before anything runs in production
- Real ongoing operational cost: upgrades, add-ons, certificates, node lifecycle
- Unsafe defaults — open pod networking, unencrypted secrets, no resource limits
- Debugging spans pod, service, ingress, DNS, policy and node layers
- YAML sprawl, and templating tools that add their own complexity
- Stateful workloads are possible but consistently harder than they look
- Easy to spend more engineering time on the platform than on the product

---

## 14. Performance Impact

| Aspect | Impact |
|---|---|
| **Container runtime** | Negligible; it is still just a process |
| **Service networking** | A small per-connection cost; IPVS or eBPF modes reduce it |
| **CPU limits** | **Throttling is the most common hidden latency cause** |
| **Scheduling** | Seconds to place a pod; longer if a node must be provisioned |
| **Node autoscaling** | Minutes — plan headroom for spikes |
| **etcd** | Sensitive to object count and churn; do not store large data in it |
| **DNS** | CoreDNS is a frequent bottleneck at high request rates; cache and tune |

> [!TIP]
> **When latency is unexplained, check CPU throttling metrics before the application.** `container_cpu_cfs_throttled_seconds_total` tells you immediately whether the kernel is pausing your process because of a limit. This one metric resolves a surprising share of "Kubernetes is slow" investigations.

---

## 15. Security Considerations

> [!CAUTION]
> **The Kubernetes API server is a cluster-wide administrative interface, and the default posture is permissive.** A service account token mounted into every pod, unrestricted pod-to-pod networking, and secrets readable by anyone with namespace access combine badly. Assume an application compromise happens, and configure the cluster so that it stays inside one namespace.

- **RBAC with least privilege**, and no `cluster-admin` for applications or CI
- **Default-deny NetworkPolicy** per namespace, with explicit allows
- **Do not mount service account tokens** into pods that do not call the API (`automountServiceAccountToken: false`)
- **Pod Security Standards (restricted)** — no privileged pods, no host network, no root
- **Encrypt etcd at rest**, and restrict who can read secrets
- **Cloud identity for pods** (IRSA, Workload Identity) rather than stored credentials
- **Scan and sign images**, and admit only signed ones via an admission policy
- **Keep the cluster upgraded** — an unpatched cluster is a large, well-documented attack surface
- **Audit logging on the API server**, shipped off-cluster

---

## 16. Mental Model

> [!NOTE]
> **Kubernetes is a thermostat, not a light switch.**
>
> You do not turn the heating on; you declare that the room should be 21°C, and a loop measures, compares and acts — forever. Pods die, nodes vanish, someone deletes something: the loop notices and corrects. This is why `kubectl delete pod` feels futile (the Deployment recreates it), why the whole system is described in YAML rather than commands, and why GitOps is such a natural fit — the repository simply becomes where the desired temperature is written down.

---

## 17. Mini Architecture Diagram

```text
   Git repo (desired state)  ──► Argo CD ──► API SERVER ──► etcd
                                                 │
                        ┌────────────────────────┼──────────────────┐
                        ▼                        ▼                  ▼
                   scheduler              controllers          admission
                        │                (reconcile loops)     (policy, signing)
   ─────────────────────┼──────────────────────────────────────────────────
        NODE 1          │            NODE 2                  NODE 3
   ┌────────────────┐   │      ┌────────────────┐      ┌────────────────┐
   │ pod api  ×2    │◄──┘      │ pod api  ×1    │      │ pod worker ×2  │
   │ requests/limits│          │                │      │                │
   │ readiness ✓    │          │ readiness ✓    │      │                │
   └───────┬────────┘          └───────┬────────┘      └────────────────┘
           │                           │
           └────────── SERVICE (stable ClusterIP + DNS) ─────────┐
                                   ▲                              │
                            INGRESS / Gateway                     ▼
                            TLS, host+path routing         NetworkPolicy:
                                   ▲                       default deny
                              cloud load balancer
                                   ▲
                                internet
```

---

## 18. Complete Request Flow

```text
CI pushes image app:<sha>; a commit updates the tag in the Git repo
    ↓
Argo CD notices the drift and applies the change to the API server
    ↓
The Deployment controller creates a new ReplicaSet
    ↓
Rolling update: one new pod at a time, old pods kept until the new one is READY
    ↓
Scheduler places the pod on a node with enough unreserved CPU and memory
    ↓
Kubelet pulls the image, starts the container, runs the startup probe
    ↓
Readiness probe passes → the pod is added to the Service's endpoints
    ↓
Only now does traffic reach it; the next old pod is terminated
    ↓
─────────────── steady state ───────────────
Ingress → Service → a healthy pod, chosen per connection
    ↓
NetworkPolicy allows api → postgres and denies everything else
    ↓
The pod reads S3 using a cloud role via IRSA — no stored credentials
    ↓
─────────────── a node dies ───────────────
Kubelet stops reporting; the node is marked NotReady
    ↓
Pods are rescheduled onto remaining nodes with capacity
    ↓
Cluster autoscaler adds a node because pods are Pending
    ↓
Capacity restored in a few minutes, with no human involved
    ↓
─────────────── a bad release ───────────────
The new version's readiness probe never passes
    ↓
The rollout STOPS — old pods are still serving, because they were not removed first
    ↓
kubectl rollout undo, or Argo CD reverts to the previous commit
    ↓
Users never saw it
    ↓
─────────────── a real incident ───────────────
Pods restart every few minutes under load
    ↓
Not a crash: exit code 137, OOMKilled — the memory limit was too low
    ↓
Raised to match the request; restarts stop
```

---

## 19. Key Takeaway

> [!IMPORTANT]
> Kubernetes is a reconciliation loop towards declared state — learn Deployment, Service, Ingress, ConfigMap and Secret, set requests and limits honestly, get probes right, apply default-deny networking, and adopt it only when your workload's complexity justifies the platform's.

---

## 20. Common Mistakes

- **No resource requests or limits**, then unexplained throttling and OOMKills
- **A liveness probe that checks a dependency**, converting a degradation into a restart storm
- **No readiness probe**, so traffic hits pods that are still starting
- **Assuming namespaces isolate the network** — they do not
- **Treating Secrets as encrypted** — they are base64
- **`cluster-admin` for CI**, or for an application service account
- **Adopting it for three services** and spending more time on the cluster than the product
- **A self-managed control plane** without a platform team
- **`latest` image tags**, making rollouts and rollbacks non-deterministic
- **Never upgrading**, until the version is unsupported and the upgrade path is a project
- **Running the primary database in-cluster** without understanding the storage implications
- **Helm charts copied from the internet** and never read
- **Debugging by `kubectl delete pod`** without asking why it was unhealthy

---

## 21. Open Source Technologies

- **Kubernetes** itself; **k3s**, **kind**, **minikube** for local and edge clusters
- **Helm**, **Kustomize** — packaging and configuration overlays
- **Argo CD**, **Flux** — GitOps reconciliation from a repository
- **Argo Rollouts**, **Flagger** — canary and progressive delivery with metric analysis
- **ingress-nginx**, **Traefik**, **Envoy Gateway** — ingress and Gateway API
- **Cilium**, **Calico** — CNI with real NetworkPolicy support (Cilium via eBPF)
- **Prometheus**, **Grafana**, **OpenTelemetry** — the observability baseline
- **External Secrets Operator**, **sealed-secrets**, **SOPS** — secret handling
- **Karpenter**, **cluster-autoscaler** — node provisioning
- **Kyverno**, **OPA Gatekeeper** — admission policy
- **k9s**, **stern**, **kubectx** — the tools that make daily operation bearable
- **Trivy**, **kube-bench**, **Polaris** — scan images, benchmarks and manifests

---

## 22. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 23. Workbook Exercise

- [ ] Find every workload without resource requests and limits. Set them on the busiest one.
- [ ] Check whether any liveness probe touches a downstream dependency.
- [ ] Look at CPU throttling metrics for your latency-sensitive service.
- [ ] Apply a default-deny NetworkPolicy in one namespace and fix what breaks.
- [ ] Count who has `cluster-admin`. Reduce the list.
- [ ] Honestly assess: would ECS Fargate or a PaaS run your workload with less effort?

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
Git → API server → etcd (desired state) → controllers reconcile → pods on nodes
                                     Ingress → Service → healthy pods
```

## 2. Request Flow

```text
Input       a declaration of what should exist
    ↓
Processing  controllers compare reality to the declaration and act, continuously
    ↓
Output      a cluster that converges on the declared state and re-converges after failure
```

## 3. Real-World Usage

Kubernetes became the industry's default platform substrate, and the durable lesson from a decade of adoption is about **fit**: it is transformative for organisations with many services and a platform team, and a persistent tax on small teams who chose it before they needed it. The failure mode is rarely the technology — it is an unowned cluster that nobody upgrades.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | A declarative control loop that runs containers across a fleet |
| **Why does it exist?** | Because placing, healing and updating containers by hand does not scale |
| **Where does it belong?** | Between your images and your machines, when there are many of both |
| **When should I use it?** | Many services, multiple teams, and someone who owns the platform |
