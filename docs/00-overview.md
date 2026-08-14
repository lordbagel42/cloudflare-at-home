# Overview

## What we're building

**lasso** (working name) is a self-hosted Cloudflare Workers platform: the Workers
programming model — fetch handlers, KV, R2, D1, Durable Objects, cron triggers,
queues, service bindings, tail logs — running on your own Kubernetes cluster, powered
by Cloudflare's open-source **workerd** runtime.

It is designed as **an application that runs on top of Kubernetes, not a Kubernetes
extension**. One Helm install brings up the whole platform. Deploying a worker is an
API call to the platform (via the `lasso` CLI or stock wrangler), not a `kubectl
apply`. There are no per-worker CRDs, Deployments, or Services; worker code and
configuration live in the platform's own database and flow to long-lived runtime
processes through dynamic worker loading. Kubernetes provides the substrate — pods,
storage, networking, restarts, scaling — and the platform manages itself on top of it,
using the Kubernetes API internally only where self-management needs it (e.g. creating
a dedicated runner pool for an isolated tenant).

The architecture deliberately mirrors Cloudflare's own production stack at homelab
scale (see [research/cloudflare-production.md](research/cloudflare-production.md)):

| Cloudflare production | lasso |
| --- | --- |
| FL front line proxy | lasso-gateway (Go reverse proxy) |
| Supervisor process (privileged, fetches code) | lasso-runner supervisor (Go, PID 1 of runner pods) |
| Workers runtime (deprivileged, seccomp'd) | stock workerd child processes |
| Isolate per script, lazily created, LRU-evicted, hard-limited | workerd process per worker version, lazily spawned, idle-reaped, heap-capped |
| Quicksilver config distribution | control-plane DB + SSE change feed + local disk cache |
| Cordons / dynamic process isolation | per-worker processes; pools per resource/trust tier |
| Immutable versions + deployments | same model, in the platform DB |
| KV/R2/D1/DO/queues backends | lasso-data services speaking workerd's native binding protocols |
| Upload API | lasso-gate (platform API + wrangler-compatible API) |

## Goals

1. **`lasso deploy` from an existing wrangler project just works** for the supported
   feature set: modules workers (JS/TS bundled, Wasm), env vars, secrets, KV, R2, D1,
   Durable Objects (incl. alarms), cron triggers, queues, service bindings, tail.
2. **Deploys are live in under 2 seconds** with no pod restarts: a deploy spawns a
   fresh workerd process for the new version, flips routing, and drains the old one —
   per-worker blue-green, invisible to every other worker.
3. **Local development is untouched**: `wrangler dev` / miniflare remain the local
   story. lasso is the *deployment target*.
4. **One Helm install, self-managing afterwards**: the platform recycles unhealthy
   runtime processes, manages memory pressure, schedules cron, retries queue
   deliveries, and (optionally, with RBAC) scales and partitions its own runner pools.
5. **Honest fidelity**: where lasso's behavior diverges from Cloudflare (limits, list
   ordering, isolation guarantees), the divergence is documented, never hidden.
6. **Continuously upgradable runtime**: workerd releases ~daily; compat dates make
   upgrades safe for user code, and a conformance suite makes them safe for the
   platform's use of experimental APIs.

## Non-goals

- **Not an edge network.** No anycast, no PoPs, no global replication. One cluster.
- **Not a hard multi-tenant boundary by default.** workerd is explicitly "not a
  hardened sandbox"; the default deployment assumes workers belong to one household of
  mutually-trusting users. Stronger isolation (dedicated pools per namespace, gVisor)
  is available as configuration, not as the default. See [06-security.md](06-security.md).
- **Not bug-for-bug Cloudflare API compatibility.** We implement the subset of the
  Cloudflare REST API that wrangler's deploy/tail/kv paths need, plus our own cleaner
  platform API. Analytics Engine, Vectorize, Workers AI, Hyperdrive, Email Workers,
  Browser Rendering, and Workflows are out of scope for v1.
- **Not a workerd fork.** Stock upstream binaries only. Anything that requires
  patching workerd is out.
- **No scale-to-zero at the pod level in v1.** Idle isolates already cost ~nothing;
  unloaded workers cost literally nothing. Pod-level scale-to-zero adds the most
  failure-prone component class in serverless (activators) for negligible homelab
  savings.

## Constraints that shaped the design

These are the load-bearing facts from research; every one is sourced in
[docs/research/](research/):

1. **workerd ships no storage and no control plane.** KV/R2/D1/queue bindings are
   client shims that speak HTTP to services *you* provide. D1 is a wrapped binding
   around an HTTP service. The platform must implement all backends.
2. **workerd config is immutable per process** (no hot reload, no admin API).
   Cloudflare's internal runtime loads code dynamically through a proprietary
   supervisor channel that OSS workerd does not expose; the OSS-faithful equivalent
   of their isolate lifecycle is therefore the *process* lifecycle — spawn lazily,
   reap when idle, replace on deploy. (workerd's `workerLoader` binding offers
   in-process dynamic loading, but it is experimental and the owner has ruled it out
   as the core mechanism; see D2.)
3. **The remaining experimental surface is small but real**:
   `service_binding_extra_handlers` (the only way to dispatch cron/queue events into
   a worker) and Durable Object `localDisk` storage. Mitigation: exact version
   pinning + a conformance suite run on every workerd upgrade.
4. **OSS workerd enforces no per-isolate limits** (`NullIsolateLimitEnforcer`).
   Process-per-worker converts this from a gap into a non-problem: per-process
   `v8Flags` heap caps + cgroups + supervisor respawn = Cloudflare's
   limit-kill-recreate behavior at process grain; the gateway owns wall-clock
   timeouts.
5. **workerd is not a hardened sandbox** (their words), and Black Hat 2026 research
   demonstrated real cross-isolate leaks in shared processes (fixed ≥ v1.20260619.1).
   Trust boundaries must be process boundaries — which process-per-worker satisfies
   by construction.
6. **Durable Objects are local to one process** with no distributed routing. Workers
   declaring DO namespaces get native config-declared DOs (SQLite `localDisk`
   storage, alarms included) and are pinned to the single-replica do pool.
7. **Prebuilt workerd binaries need glibc 2.35+** (no Alpine/musl), linux amd64/arm64.
8. **Nothing like this exists for Kubernetes.** wdl (single-region,
   docker-compose-shaped) is the closest existing system; it took the
   dynamic-loading path we rejected, and remains useful as prior art for the
   control-plane and failover designs.

## Glossary

| Term | Meaning |
| --- | --- |
| **workerd** | Cloudflare's open-source Workers runtime (C++/V8). One process hosts many isolates. |
| **worker** | A unit of user code + config (bindings, compat date) deployed to lasso. Addressed as `namespace/name`. |
| **namespace** | lasso's tenant/project grouping (like a Cloudflare account+environment). Not a Kubernetes namespace. |
| **version** | An immutable upload of a worker: modules + metadata. Identified by a ULID + content hash. Never mutated, never reused. |
| **deployment** | The pointer that says which version of a worker serves traffic. Rollback = repoint. |
| **bundle** | The stored artifact of a version: module files + `metadata.json` (mirrors Cloudflare's multipart upload metadata). |
| **worker process** | One workerd OS process running exactly one worker version (plus its harness worker), listening on a unix socket, managed by the supervisor. |
| **harness worker** | The small trusted platform worker co-declared in every worker process's config: receives cron/queue dispatches (via `service_binding_extra_handlers`), forwards tail events, serves bundled assets. |
| **runner** | A pod = lasso supervisor (Go, PID 1) + N workerd child processes. Belongs to a pool. |
| **pool** | A Deployment of identical runners forming one resource domain (default: `user` pool; optional dedicated pools per namespace; `do` pool for Durable Objects). |
| **lasso-gate** | The control plane: API server, DB, bundle store, scheduler, auth. |
| **lasso-gateway** | The data-plane ingress proxy: hostname → worker → runner. |
| **lasso-data** | Storage backends for KV/R2/D1/queues/cache. |
| **binding** | A named capability on a worker's `env` (KV namespace, R2 bucket, D1 DB, DO namespace, queue producer, service binding, secret, var), declared in the process's workerd config. |

## Repository layout (planned)

This will be a monorepo:

```
cloudflare-at-home/
├── docs/                  # this plan, then living design docs
├── gate/                  # lasso-gate control plane (Go)
├── gateway/               # lasso-gateway ingress proxy (Go)
├── runner/
│   ├── supervisor/        # Go supervisor (PID 1): process manager, router, config renderer
│   └── harness/           # harness worker (TypeScript): dispatch, tail, assets
├── data/                  # lasso-data storage services (Go)
├── cli/                   # lasso CLI (TypeScript)
├── proto/                 # internal API definitions (OpenAPI/proto) + bundle format schema
├── deploy/
│   ├── helm/lasso/        # the Helm chart
│   └── kind/              # kind-based dev/e2e environment
├── conformance/           # workerd-upgrade conformance suite (wd-test + e2e)
└── hack/                  # dev scripts
```

Languages: **Go** for all long-running services (gate, gateway, supervisor, data),
**TypeScript** for everything that touches the Workers ecosystem (harness worker,
binding stubs, CLI). Version-pinned **workerd** binaries from npm/GitHub releases.
