# Architecture

## Components

```
                        ┌────────────────────────────────────────────────────────┐
                        │                    Kubernetes cluster                  │
                        │                                                        │
   users ──HTTPS──▶ k8s Gateway (Cilium/Envoy/Traefik, wildcard TLS)             │
                        │        │                                               │
     *.workers.lan ─────┼────────┤ HTTPRoute (static, from Helm)                 │
     api.lasso.lan ─────┼──┐     │                                               │
                        │  │     ▼                                               │
                        │  │  ┌──────────────┐   route table (SSE)  ┌─────────┐  │
                        │  │  │ lasso-gateway│◀────────────────────▶│         │  │
                        │  │  │  (Go proxy)  │                      │  lasso- │  │
                        │  │  └──────┬───────┘                      │  gate   │  │
                        │  │         │ HTTP + X-Lasso-Worker header │ (control│  │
                        │  └─────────┼─────────────────────────────▶│  plane) │  │
                        │            ▼                              │         │  │
                        │  ┌─────────────────────┐   bundles (HTTP) │ SQLite  │  │
                        │  │ user pool (Deploy)  │◀────────────────▶│ + blob  │  │
                        │  │ ┌─────────────────┐ │                  │ store   │  │
                        │  │ │ runner pod      │ │   cron/queue     │ on PVC  │  │
                        │  │ │ ┌─────────────┐ │ │   dispatch       └────┬────┘  │
                        │  │ │ │ supervisor  │ │ │◀───────────────────── │       │
                        │  │ │ │  (Go, PID1) │ │ │                       │       │
                        │  │ │ └──────┬──────┘ │ │                  k8s API      │
                        │  │ │        │spawns  │ │             (self-management, │
                        │  │ │ ┌──────▼──────┐ │ │              optional RBAC)   │
                        │  │ │ │   workerd   │ │ │                               │
                        │  │ │ │ ┌─────────┐ │ │ │  binding RPC   ┌───────────┐  │
                        │  │ │ │ │ loader  │─┼─┼─┼───────────────▶│ lasso-data│  │
                        │  │ │ │ │ worker  │ │ │ │  (KV/R2/D1/    │ (Go, PVC) │  │
                        │  │ │ │ ├─────────┤ │ │ │   queues)      └───────────┘  │
                        │  │ │ │ │ user    │ │ │ │                               │
                        │  │ │ │ │ isolates│ │ │ │  DO calls      ┌───────────┐  │
                        │  │ │ │ └─────────┘ │ │ │───────────────▶│ do pool   │  │
                        │  │ │ └─────────────┘ │ │                │(StatefulSet│ │
                        │  │ └─────────────────┘ │                │ workerd + │  │
                        │  └─────────────────────┘                │ facets+PVC│  │
                        │                                         └───────────┘  │
                        └────────────────────────────────────────────────────────┘
```

Six deployables, one Helm chart:

| Component | Kind | Language | State |
| --- | --- | --- | --- |
| **lasso-gate** | Deployment (1 replica v1) | Go | SQLite + content-addressed blob store on PVC |
| **lasso-gateway** | Deployment (1–N replicas) | Go | none (route table cached in memory) |
| **runner pools** | Deployment per pool (`user` pool by default, more via self-management) | Go supervisor + workerd + TS loader worker | none (disk = bundle cache only) |
| **do pool** | StatefulSet (1 replica v1) | same runner image, DO-mode config | DO SQLite databases on PVC |
| **lasso-data** | Deployment (1 replica v1) | Go | KV/R2/D1/queue data on PVC |
| **(optional) lasso-ui** | Deployment | TS | none — talks to gate API (post-v1) |

Everything speaks plain HTTP inside the cluster (ClusterIP services); internal calls
carry a shared-secret bearer token provisioned by the chart (see
[06-security.md](06-security.md)).

## Identity model

```
namespace  "blog"                         (tenant/project; default ns: "default")
└── worker "api"                          (FQN: blog/api)
    ├── version 01JD…X4 (immutable)       bundle: modules + metadata.json
    ├── version 01JE…9Q (immutable)
    └── deployment → 01JE…9Q              (the active pointer; rollback = repoint)
```

- **Worker id** used everywhere internally: `blog/api`.
- **Loader isolate id**: `blog/api@01JE…9Q` — namespace, worker, *and version*. Because
  the id is immutable and version-scoped, deploying never mutates a loaded worker: the
  new version is a new id, cold-loaded on first request; the old id is evicted lazily.
  This is the keystone invariant of the whole platform (borrowed from wdl and from
  Cloudflare's versions/deployments model).
- **Default URL**: `https://api.blog.workers.<base-domain>`; workers in the `default`
  namespace also get `https://api.workers.<base-domain>`. Custom hostnames map to a
  worker via the routes table in gate.

## Data flows

### Deploy (CLI → live), target < 2 s

1. `lasso deploy` bundles the project with esbuild (same module rules as wrangler),
   producing modules + `metadata.json` (Cloudflare multipart-metadata-shaped).
2. CLI PUTs the multipart bundle to gate (`/v1/namespaces/blog/workers/api/versions`).
3. Gate validates (size caps, env-size preflight against workerd's 1 MB env budget,
   binding references exist, compat date ≤ pinned workerd version), stores modules in
   the content-addressed blob store, inserts an immutable `version` row, and — unless
   `--no-activate` — updates the `deployment` pointer in the same transaction.
4. Gate pushes a route-table delta on the SSE change feed. Gateways apply it in
   memory (< 100 ms typical).
5. Next request for `api.blog.workers.lan` is forwarded with
   `X-Lasso-Worker: blog/api@01JE…9Q`. The runner's loader worker misses its cache,
   fetches the bundle envelope from gate (supervisor-mediated disk cache), calls
   `env.LOADER.get("blog/api@01JE…9Q", cb)`, and serves. Isolate cold start is
   single-digit milliseconds; bundle fetch is intra-cluster.

No pods are created, restarted, or reconfigured. Kubernetes is not on the deploy path.

### HTTP request

1. Cluster Gateway (TLS) → lasso-gateway with original `Host`.
2. Gateway strips any client-supplied `X-Lasso-*` headers, resolves
   `Host` (+ optional path routes) → `(worker, active version, pool)` from its
   in-memory route table, applies per-worker wall-clock timeout (default 30 s) and
   in-flight cap, picks a runner pod (rendezvous hash on worker id for isolate-cache
   affinity, falling back to any healthy pod), and proxies with
   `X-Lasso-Worker: ns/name@version`.
3. Runner workerd socket → loader worker `fetch()`: looks up/loads the version stub,
   calls `stub.getEntrypoint().fetch(request)` with `request.cf`-equivalent metadata,
   returns the response. WebSockets pass through end to end.
4. Tail events (logs/exceptions/subrequests) stream from the attached platform tail
   worker to gate (see [07-observability.md](07-observability.md)).

### Cron / queue delivery

Gate owns the schedules (cron table) and lasso-data owns queue storage. Both dispatch
the same way: POST to a runner's internal dispatch endpoint; the loader worker calls
`stub.getEntrypoint().scheduled(...)` / `.queue(...)` (requires the
`service_binding_extra_handlers` compat flag on the *loader*, which is experimental —
accepted and conformance-tested). Outcomes (`ok`/`exception`/retryAll…) return in the
response envelope; gate/data apply retry policy and dead-letter queues.

### Durable Objects

User workers get a `DurableObjectNamespace`-shaped binding stub. Calls route to the
**do pool** (single StatefulSet replica in v1), where a config-declared supervisor DO
(SQLite `localDisk` storage on the PVC) instantiates the user's DO class as a
**facet**, keyed by namespace/class/object id. Object identity is guaranteed by
routing all DO traffic for a given namespace to the one do-pool pod — the platform's
substitute for workerd's missing distributed DO routing. Alarms run natively inside
the facet's storage. Details in [03-bindings.md](03-bindings.md).

## Self-management (the Kubernetes-facing side)

The platform treats Kubernetes as its machine manager, through a narrow, optional
`k8s` module in gate (in-cluster ServiceAccount, RBAC scoped to lasso's own release
namespace):

| Concern | Mechanism |
| --- | --- |
| Runner crash / wedge | Supervisor exits nonzero → kubelet restarts container. Liveness = supervisor HTTP; readiness = workerd control-fd handshake + loader self-check. |
| Memory pressure | cgroup `memory.max` per pod (backstop) + supervisor RSS watchdog: over soft threshold → mark NotReady, drain (gateway stops picking it), restart workerd child cleanly, resume. Loaded isolates are cache, not state — recycling is always safe. |
| Version eviction | Loader tracks loaded ids; on activating a new version it aborts sibling old-version isolates (`abortIsolate` pattern, conformance-tested). Process recycling is the fallback GC. |
| Scaling pools | v1: static `replicas` in Helm values + optional HPA manifest. v1.5: gate can resize pool Deployments via the k8s API from observed concurrency. |
| Dedicated pools (trust domains) | Namespace flagged `isolation: dedicated` → gate creates/labels a separate runner Deployment (same image/config, different pool name) and routes that namespace's traffic to it. This is the Cloudflare "cordon" idea, driven by the platform itself. |
| workerd upgrades | New runner image tag in Helm → rolling restart (`maxUnavailable: 0`); loaded workers cold-load again on the new pods — safe by the immutable-version invariant + compat dates. Conformance suite gates the image bump in CI. |
| Degraded mode | Without RBAC (or outside k8s entirely, e.g. docker-compose for dev), everything works except dedicated-pool creation and autoscaling — those degrade to log warnings. The platform never *requires* the k8s API for the data path. |

## What runs where (trust map)

| Process | Trust | Holds |
| --- | --- | --- |
| lasso-gate | trusted | DB, secrets (encrypted at rest), master keys, k8s ServiceAccount |
| lasso-gateway | trusted | route table, internal token |
| runner supervisor | trusted | internal token, bundle disk cache |
| workerd loader worker | trusted (platform-authored JS) | capability stubs to data services; per-load bundle envelopes |
| user isolates | **untrusted** | only their own `env` stubs (props-scoped), `globalOutbound` restricted to public network |
| lasso-data | trusted | user data at rest |
| do pool | semi-trusted (runs user DO code) | DO storage PVC; same pool/pod isolation rules as runners |

The full threat model and the "when is shared-pool acceptable" discussion live in
[06-security.md](06-security.md).

## Key decisions (summaries — rationale in [10-decisions.md](10-decisions.md))

1. **Application on k8s, not operator/CRDs** (D1). Workers are DB rows; deploys are
   API calls; k8s objects exist only for the platform's own components.
2. **Dynamic loading via `workerLoader`, not config-per-worker** (D2). Deploys without
   restarts; isolate-level multi-tenancy inside pools; accepts the experimental-flag
   risk with pinning + conformance.
3. **Custom Go supervisor wrapping stock workerd; miniflare as reference only** (D3).
4. **Bindings via `ctx.exports` loopback stubs + props** (D4) — nothing privileged in
   user `env` that needs stripping; scoping enforced by props the user code can't see
   or forge.
5. **Trust domain = process (pool), mirroring Cloudflare's fallback posture** (D5).
6. **Go for services, TypeScript for Workers-ecosystem code** (D6).
7. **SQLite-on-PVC for control plane and data services in v1; interfaces designed for
   Postgres/S3 swap-in later** (D7).
8. **Wrangler-compatible API is a compatibility *layer* on gate, not the native API**
   (D8).
9. **Gateway API (HTTPRoute) for ingress; static routes only** (D9).
10. **Cache API and static assets: minimal v1** (D10) — cache = passthrough no-op
    initially (honest miss), assets = served by loader from the bundle.
