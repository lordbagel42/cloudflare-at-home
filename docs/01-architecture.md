# Architecture

> Revised 2026-08-14: the runner no longer uses `workerLoader` dynamic loading. User
> workers run as **supervisor-managed workerd processes with static per-worker
> config** — one process per active worker version — mirroring Cloudflare's
> supervisor/runtime split and their process-isolation posture. See D2/D16 in
> [10-decisions.md](10-decisions.md) for the rationale and the rejected design.

## Components

```
                        ┌────────────────────────────────────────────────────────────┐
                        │                      Kubernetes cluster                    │
                        │                                                            │
   users ──HTTPS──▶ k8s Gateway (Cilium/Envoy/Traefik, wildcard TLS)                 │
                        │        │                                                   │
     *.workers.lan ─────┼────────┤ HTTPRoute (static, from Helm)                     │
     api.lasso.lan ─────┼──┐     │                                                   │
                        │  │     ▼                                                   │
                        │  │  ┌──────────────┐    route feed (SSE)   ┌─────────┐     │
                        │  │  │ lasso-gateway│◀────────────────────▶│         │     │
                        │  │  │  (Go proxy)  │                      │  lasso- │     │
                        │  │  └──────┬───────┘                      │  gate   │     │
                        │  │         │ X-Lasso-Worker: ns/name@ver  │ (control│     │
                        │  └─────────┼─────────────────────────────▶│  plane) │     │
                        │            ▼                              │ SQLite+ │     │
                        │  ┌───────────────────────────┐   bundles  │ blobs   │     │
                        │  │ user pool (Deployment)    │◀──────────▶│ on PVC  │     │
                        │  │ ┌───────────────────────┐ │            └────┬────┘     │
                        │  │ │ runner pod            │ │  cron/queue     │          │
                        │  │ │ ┌───────────────────┐ │ │  dispatch       │          │
                        │  │ │ │ supervisor (Go)   │ │ │◀────────────────┘          │
                        │  │ │ │  · route/spawn    │ │ │                            │
                        │  │ │ │  · drain/reap     │ │ │            k8s API         │
                        │  │ │ └───┬───────┬───────┘ │ │       (self-management,    │
                        │  │ │     │uds    │uds      │ │        optional RBAC)      │
                        │  │ │ ┌───▼───┐ ┌─▼─────┐   │ │                            │
                        │  │ │ │workerd│ │workerd│ … │ │  native binding shims      │
                        │  │ │ │blog/  │ │dash/  │   │ │  (KV/R2/D1/queue/cache     │
                        │  │ │ │api@v9 │ │app@v2 │   │ │   HTTP protocols)          │
                        │  │ │ │+harness│ │+harness│  │ │        │                   │
                        │  │ │ └───────┘ └───────┘   │ │        ▼                   │
                        │  │ └───────────────────────┘ │   ┌───────────┐            │
                        │  └───────────────────────────┘   │ lasso-data│            │
                        │                                  │ (Go, PVC) │            │
                        │  ┌───────────────────────────┐   └───────────┘            │
                        │  │ do pool (StatefulSet+PVC) │                            │
                        │  │  same runner image; hosts │                            │
                        │  │  workers that declare DOs │                            │
                        │  │  (native namespaces,      │                            │
                        │  │   SQLite localDisk)       │                            │
                        │  └───────────────────────────┘                            │
                        └────────────────────────────────────────────────────────────┘
```

Six deployables, one Helm chart:

| Component | Kind | Language | State |
| --- | --- | --- | --- |
| **lasso-gate** | Deployment (1 replica v1) | Go | SQLite + content-addressed blob store on PVC |
| **lasso-gateway** | Deployment (1–N) | Go | none (route table cached, snapshot on emptyDir) |
| **runner pools** | Deployment per pool (`user` default; more via self-management) | Go supervisor + N × pinned workerd processes | none (disk = bundle/module cache) |
| **do pool** | StatefulSet (1 replica v1) | same runner image | DO SQLite databases on PVC |
| **lasso-data** | Deployment (1 replica v1) | Go | KV/R2/D1/queue data on PVC |
| **(optional) lasso-ui** | Deployment | TS | none (post-v1) |

Everything speaks plain HTTP in-cluster; internal calls carry a chart-generated
bearer token ([06-security.md](06-security.md)).

## The Cloudflare correspondence

This is the same division of labor as Cloudflare's edge, one worker per runtime
process instead of thousands of isolates per process (their density trick — the
proprietary isolate limiter/scheduler — is the part OSS doesn't ship; process
granularity is their own documented fallback for code that needs real limits):

| Cloudflare production | lasso |
| --- | --- |
| FL front line | lasso-gateway |
| Supervisor (privileged: fetches code, holds keys) | runner supervisor (Go, PID 1) |
| Runtime process per cordon; per-worker quarantine processes | one workerd process per active worker version |
| Isolate created lazily on first request; LRU-evicted | worker process spawned lazily on first request; idle-reaped (LRU) |
| 128 MB isolate heap limit | per-process `v8Flags` heap cap + cgroup |
| Quicksilver push + supervisor lazy load | gate route feed (SSE) + supervisor bundle cache |
| Immutable versions, deployments pointer | identical model in gate's DB |
| Upload API → seconds to global | upload API → next request runs new version |

## Identity model

```
namespace  "blog"                         (tenant/project; default ns: "default")
└── worker "api"                          (FQN: blog/api)
    ├── version 01JD…X4 (immutable)       bundle: modules + metadata.json
    ├── version 01JE…9Q (immutable)
    └── deployment → 01JE…9Q              (the active pointer; rollback = repoint)
```

- **Process identity = `ns/name@version`.** A worker process only ever runs one
  immutable version; a deploy means a *new process*, never a mutated one. Old and new
  can coexist during drain (true per-worker blue-green). This keeps the keystone
  invariant from the previous design — immutable version-scoped runtime identity —
  while moving it from isolate ids to process identities.
- **Default URL**: `https://api.blog.workers.<base-domain>`; `default`-namespace
  workers also get `https://api.workers.<base-domain>`. Custom hostnames map via
  gate's routes table.

## Data flows

### Deploy (CLI → live), target < 2 s

1. `lasso deploy` bundles with esbuild → modules + `metadata.json` (Cloudflare
   multipart-metadata-shaped).
2. CLI PUTs multipart to gate; gate validates (size caps, binding refs, compat date ≤
   pinned workerd, reserved names), stores blobs, inserts the immutable `version`,
   flips the `deployment` pointer (same transaction unless `--no-activate`).
3. Gate emits a route-feed delta. Gateways update their maps; **runner supervisors
   that currently host the worker prewarm the new version**: fetch bundle, write
   modules to tmpfs, render per-worker capnp config, spawn workerd, wait for
   control-fd ready.
4. Supervisors atomically flip local routing `ns/name → @newversion`, drain the old
   process (grace period for in-flight/WebSockets), then reap it.
5. Pods not hosting the worker do nothing; their first request for it cold-spawns.

Kubernetes is not on the deploy path; no pods are created or restarted. The unit of
replacement is one worker's own process.

### HTTP request

1. Cluster Gateway (TLS) → lasso-gateway.
2. Gateway strips client `X-Lasso-*` headers, resolves Host → `(worker, active
   version, pool)`, applies wall-clock timeout + in-flight caps, picks a pod
   (rendezvous hash on worker id → warm-process affinity), proxies with
   `X-Lasso-Worker: blog/api@01JE…9Q`.
3. The pod supervisor routes to the matching child process's unix socket — spawning
   it first if absent (cold start = bundle fetch [cached] + workerd process start).
4. Inside the process, the socket routes straight to the user worker (harness worker
   only fronts internal dispatch paths). Response streams back; WebSockets pass
   through end to end.

### Cron / queue delivery

Gate schedules crons; lasso-data leases queue batches. Both POST dispatches to a
runner (`/__lasso/dispatch/*`, supervisor-authenticated). The supervisor routes the
dispatch into the worker process, where the co-declared **harness worker** — the only
platform JS in the process — calls `scheduled()` / `queue()` on its service binding
to the user worker (compat flag `service_binding_extra_handlers`, held by the harness
only) and returns the outcome envelope; gate/data apply retry/DLQ policy.

### Durable Objects

Workers whose metadata declares DO namespaces are assigned to the **do pool**. Their
per-worker config declares the namespaces natively (`durableObjectNamespaces` with
platform-generated stable `uniqueKey`s, `enableSql`, `durableObjectStorage =
localDisk` on the PVC) — no facets, no dynamic-loading indirection; DOs, storage, and
alarms all behave exactly as workerd implements them. Object uniqueness holds because
the do pool has one replica and the gateway routes those workers only there. DO
version-skew on deploy follows the process-swap rule: in-flight objects live in the
old process until drain completes; the new process serves new requests (divergence
from CF's finer-grained rules is documented in [03-bindings.md](03-bindings.md)).

## Self-management (the Kubernetes-facing side)

| Concern | Mechanism |
| --- | --- |
| Worker process crash | Supervisor respawns with backoff; repeated crashes → worker marked failing in gate (visible in `lasso list`/tail), backoff caps blast radius to that worker. |
| Pod crash / wedge | kubelet restarts; supervisor state is rebuilt from route feed + lazy spawns. |
| Memory | Per-process: `v8Flags` heap cap (default 192 MB) makes a leaking worker abort *its own* process, which the supervisor respawns — Cloudflare's evict-and-recreate at process grain. Pod cgroup is the backstop; RSS watchdog reaps the worst offender first instead of letting the OOM killer choose. |
| Idle workers | LRU process reaping after idle TTL (default 15 m; DO-pool exempt by default). Warm again on next request. |
| Scaling pools | v1: Helm replicas + optional HPA. v1.5: gate resizes pool Deployments from gateway concurrency metrics. |
| Dedicated pools | Namespace `isolation: dedicated` → gate creates a labeled runner Deployment for it and routes accordingly (cordons, platform-driven). |
| workerd upgrades | New runner image (conformance-gated in CI) → rolling pods (`maxUnavailable: 0`); worker processes cold-spawn on new pods; compat dates protect user code. |
| Degraded mode | Without RBAC / outside k8s (compose dev mode): everything except dedicated pools + autoscaling works; those log warnings. The data path never requires the k8s API. |

## What runs where (trust map)

| Process | Trust | Holds |
| --- | --- | --- |
| lasso-gate | trusted | DB, encrypted secrets, master key, k8s ServiceAccount |
| lasso-gateway | trusted | route table, internal token |
| runner supervisor | trusted | internal token, bundle/module cache, child lifecycle |
| harness worker (per worker process) | trusted platform JS | dispatch entrypoint, tail forwarding, asset serving |
| user worker process | **untrusted** | its own bindings only (config-scoped), `globalOutbound` = public-only |
| lasso-data | trusted | user data at rest |
| do pool processes | untrusted user code | their own DO storage directories |

Note the isolation upgrade over the previous design: **every worker is its own
process** even in the shared pool — the V8 isolate boundary is no longer relied on
between tenants at all. Pools now group processes for *resource* management (cgroup,
gVisor, scheduling), not for code isolation.

## Key decisions (summaries — rationale in [10-decisions.md](10-decisions.md))

1. **Application on k8s, not operator/CRDs** (D1).
2. **Process-per-worker-version, supervisor-managed; no `workerLoader`** (D2 revised
   + D16). Deploys = per-worker blue-green process swaps; lazy spawn + LRU reap
   recreate Cloudflare's isolate lifecycle at process grain.
3. **Custom Go supervisor wrapping stock workerd; miniflare as reference only** (D3).
4. **Native workerd binding shims** — kvNamespace/r2Bucket/D1-wrapped/queue/cache
   declared in per-worker config, pointed at lasso-data; scoping via per-binding
   injected headers (D4 revised).
5. **Trust domain = process; pools are resource domains** (D5 revised).
6. **Go for services, TypeScript for Workers-ecosystem code** (D6).
7. **SQLite-on-PVC v1; Postgres/S3 interfaces reserved** (D7).
8. **Native platform API first; wrangler-compat as a layer** (D8).
9. **Gateway API, static routes; per-worker routing inside lasso-gateway** (D9).
10. **Honest degradation for Cache/assets v1** (D10).
