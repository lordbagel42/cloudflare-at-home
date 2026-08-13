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
| Workers runtime (deprivileged, seccomp'd) | stock workerd child process |
| Dispatch namespaces / Worker Loader | loader worker + `workerLoader` binding |
| Quicksilver config distribution | control-plane DB + SSE change feed + local disk cache |
| Cordons (process per trust tier) | runner pools (Deployment per trust domain) |
| Immutable versions + deployments | same model, in the platform DB |
| KV/R2/D1/DO/queues backends | lasso-data services over PVCs |
| Upload API | lasso-gate (platform API + wrangler-compatible API) |

## Goals

1. **`lasso deploy` from an existing wrangler project just works** for the supported
   feature set: modules workers (JS/TS bundled, Wasm), env vars, secrets, KV, R2, D1,
   Durable Objects (incl. alarms), cron triggers, queues, service bindings, tail.
2. **Deploys are live in under 2 seconds** with no pod restarts (dynamic loading; the
   version id changes, the next request cold-loads the new isolate in milliseconds).
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
2. **workerd config is immutable per process** (no hot reload, no admin API), *but*
   the `workerLoader` binding loads complete workers at runtime, keyed by immutable
   ids, with capability-scoped `env`. This is how deploys avoid restarts.
3. **`workerLoader` is `--experimental` in OSS workerd** (as is
   `service_binding_extra_handlers`, needed to dispatch cron/queue events into loaded
   workers). Mitigation: exact version pinning + a conformance suite run on every
   workerd upgrade, and the abstraction kept thin enough to swap strategies if
   upstream breaks it.
4. **OSS workerd enforces no per-isolate limits** (`NullIsolateLimitEnforcer`; the
   `limits` API fields are parsed and ignored) and **never evicts loaded workers on
   its own**. The supervisor owns memory: cgroup limits, RSS watchdog, process
   recycling; the gateway owns wall-clock timeouts.
5. **workerd is not a hardened sandbox** (their words), and Black Hat 2026 research
   demonstrated real cross-isolate leaks in shared processes (fixed ≥ v1.20260619.1).
   Trust boundaries must be process boundaries.
6. **Durable Objects are local to one process** and dynamically loaded workers can't
   declare DO namespaces — DOs run via the **facets** API under a config-declared
   supervisor DO, with SQLite-on-disk storage. DO traffic must route to the single
   owning process.
7. **Prebuilt workerd binaries need glibc 2.35+** (no Alpine/musl), linux amd64/arm64.
8. **Nothing like this exists for Kubernetes.** wdl (single-region,
   docker-compose-shaped) is the closest system and validates the loader-worker
   architecture end to end.

## Glossary

| Term | Meaning |
| --- | --- |
| **workerd** | Cloudflare's open-source Workers runtime (C++/V8). One process hosts many isolates. |
| **worker** | A unit of user code + config (bindings, compat date) deployed to lasso. Addressed as `namespace/name`. |
| **namespace** | lasso's tenant/project grouping (like a Cloudflare account+environment). Not a Kubernetes namespace. |
| **version** | An immutable upload of a worker: modules + metadata. Identified by a ULID + content hash. Never mutated, never reused. |
| **deployment** | The pointer that says which version of a worker serves traffic. Rollback = repoint. |
| **bundle** | The stored artifact of a version: module files + `metadata.json` (mirrors Cloudflare's multipart upload metadata). |
| **loader worker** | The trusted, platform-authored worker running in every runner's workerd; holds the `workerLoader` binding, fetches bundles, builds `env`, dispatches events into user workers. |
| **runner** | A pod = lasso supervisor (Go, PID 1) + stock workerd child. Belongs to a pool. |
| **pool** | A Deployment of identical runners forming one trust domain (default: `user` pool; optional dedicated pools per namespace; `do` pool for Durable Objects). |
| **lasso-gate** | The control plane: API server, DB, bundle store, scheduler, auth. |
| **lasso-gateway** | The data-plane ingress proxy: hostname → worker → runner. |
| **lasso-data** | Storage backends for KV/R2/D1/queues/cache. |
| **binding** | A named capability on a worker's `env` (KV namespace, R2 bucket, D1 DB, DO namespace, queue producer, service binding, secret, var). |
| **facet** | workerd's mechanism for running a dynamically-loaded DO class under a config-declared supervisor DO, with its own storage. |

## Repository layout (planned)

This will be a monorepo:

```
cloudflare-at-home/
├── docs/                  # this plan, then living design docs
├── gate/                  # lasso-gate control plane (Go)
├── gateway/               # lasso-gateway ingress proxy (Go)
├── runner/
│   ├── supervisor/        # Go supervisor (PID 1)
│   └── loader/            # loader worker + binding stubs (TypeScript)
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
**TypeScript** for everything that touches the Workers ecosystem (loader worker,
binding stubs, CLI). Version-pinned **workerd** binaries from npm/GitHub releases.
