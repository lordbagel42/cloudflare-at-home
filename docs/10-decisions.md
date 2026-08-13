# Decision record

ADR-style. Each entry: decision, context, alternatives rejected, consequences.
Status of all: **accepted** (2026-08-13) unless noted.

## D1 — Application on Kubernetes, not an operator/CRDs

Workers are rows in the platform DB; deploying talks to the platform API. Kubernetes
objects exist only for lasso's own components.
**Context:** explicit product direction from the owner ("I just want a service
intended to be running on top of Kubernetes… a thin custom control plane"); plus:
deploy latency shouldn't ride etcd/reconcile loops, worker count shouldn't bloat
etcd, and kubectl is the wrong UX for function deploys.
**Rejected:** CRD-per-worker operator (original draft design — see
research/kubernetes-patterns.md for its record); CRDs as an *additional* GitOps
surface (post-v1 candidate: a tiny optional controller that syncs `Worker` CRs to the
API — explicitly deferred, not designed).
**Consequences:** platform owns its own auth, audit, and API versioning; GitOps
users drive the CLI from CI instead of manifests; gains: sub-second deploys, no
etcd coupling, portable off k8s (compose dev mode).

## D2 — Dynamic loading (`workerLoader`) instead of config-per-worker processes

One long-lived workerd per runner with a trusted loader worker; user code loaded at
runtime keyed by immutable `ns/name@version` ids.
**Context:** workerd config is start-time-only; restarts per deploy would mean pod
churn and cold caches. `workerLoader` is how Cloudflare's own Workers-for-Platforms
model maps to OSS; wdl proves it in production shape.
**Rejected:** capnp config generation + process restart per deploy (the selflare
model — simple but restart-per-deploy, no isolate cache); process-per-worker
(clean isolation but heavy for many small workers — reserved as the *dedicated pool*
option, and as fallback strategy if upstream breaks the loader API).
**Consequences:** accept `--experimental` risk → exact pinning + conformance suite
(WP7.4); platform owns eviction and memory pressure (supervisor watchdog).

## D3 — Custom Go supervisor around stock workerd; miniflare as reference only

**Context:** owner preference for a custom wrapper; miniflare is dev-oriented (Node
in the data path, unstable internal APIs, single-node assumptions); Cloudflare's own
production supervisor/runtime split is the proven shape and is documented enough to
imitate.
**Rejected:** embedding miniflare as the per-pod runtime host; forking workerd
(never).
**Consequences:** we implement bundle caching, readiness, watchdogs ourselves
(~small Go surface); we read miniflare source freely for wire-protocol reference
(kv/r2/d1 shapes) without depending on it at runtime.

## D4 — Bindings as `ctx.exports` loopback stubs with props; storage in Go services

**Context:** dynamic workers' `env` accepts Fetcher/RPC stubs; props are
set-at-creation, invisible and unforgeable from user code — a clean capability
scoping primitive. wdl's experience: putting raw privileged fetchers in env forces a
wrapper/stripping layer (complexity + risk); loopback stubs avoid it entirely.
**Rejected:** workerd-native `kvNamespace`/`r2Bucket` binding shims pointed at
external services (only work for config-declared workers, not dynamic ones);
vendoring miniflare simulator workers (ties us to `miniflare:shared` internals).
**Consequences:** we author the binding API-shape classes in TS (KV/R2/D1/DO/queue)
and their conformance tests; storage services stay in Go where operational tooling
is better.

## D5 — Trust domain = process/pool; default one shared user pool

**Context:** OSS workerd enforces no isolate limits and isn't a hardened sandbox
(upstream statement + Black Hat 2026 findings); Cloudflare's own answer for
suspicious/untrusted code is process isolation. Homelab default is one household.
**Rejected:** pretending isolate boundaries are a security guarantee; defaulting to
process-per-worker (resource waste at default scale).
**Consequences:** `isolation: dedicated` namespaces get own pools via
self-management; per-isolate CPU/heap limits documented as unavailable; gateway
wall-clock timeout is the request-level backstop.

## D6 — Go for services, TypeScript for Workers-ecosystem code

**Context:** Go: single-binary services, first-class k8s clients, easy static
distroless images. TS: the loader *must* be a worker; binding classes and CLI live in
the Workers/wrangler ecosystem.
**Rejected:** Rust everywhere (fine choice, smaller agent familiarity, no ecosystem
pull for the CLI); Node for gate/gateway (weaker fit for the proxy/scheduler and for
low-RSS homelab targets).

## D7 — SQLite-on-PVC everywhere in v1; interfaces reserved for Postgres/S3

**Context:** homelab scale (≤ tens of rps, ≤ hundreds of workers) is far inside
SQLite's envelope; one moving part fewer than Postgres; blob store dedups modules.
**Rejected for v1:** Postgres (operational surface), embedded KV stores like
Badger (SQLite's SQL + tooling wins for inspectability), object storage dependency
(MinIO) as a hard requirement.
**Consequences:** single-replica gate/data in v1 (accepted); storage interfaces
(`gate/store`, `data/store`) written against interfaces so Postgres/S3 backends are
additive; Litestream hooks in chart for durability.

## D8 — Native platform API first; wrangler-compat as a translation layer (M7)

**Context:** cloning the CF API as the *primary* surface contorts identity (account
ids, zones) and error semantics; but wrangler compatibility is a huge adoption lever
and wdl proves the subset is tractable.
**Rejected:** compat-API-only (wdl's approach); no compat at all.
**Consequences:** one internal ingest path, two front doors; compat gaps fail loud
with structured errors.

## D9 — Gateway API with static routes; per-worker routing inside lasso-gateway

**Context:** ingress-nginx is retired (2026); Gateway API is GA. Per-worker
HTTPRoutes would reintroduce k8s-object-per-worker (violates D1) and add propagation
latency; a wildcard host route + in-process routing is faster and simpler.
**Rejected:** per-worker HTTPRoutes; Ingress API; lasso-gateway implemented *as a
workerd worker* (wdl does this — elegant, but a Go proxy gives us timeouts, retries,
EndpointSlice awareness, and metrics with far less exotic code).

## D10 — Honest degradation for Cache API and assets in v1

Cache: no-op passthrough (workerd's own default without `cacheApiOutbound`); assets:
bundle-embedded. Never fake semantics; upgrade paths specified (03-bindings.md).
**Rejected:** shipping a half-correct cache that changes response semantics.

## D11 — Queue delivery loops live in lasso-data; cron in gate

**Context:** queues need lease/retry state colocated with message storage; cron
needs the config DB and is naturally single-writer.
**Rejected:** a separate scheduler service (more pods for no isolation win at this
scale).

## D12 — CLI wraps wrangler's `unstable_readConfig` + own esbuild pipeline

**Context:** config parsing parity matters more than bundling parity;
`unstable_getMiniflareWorkerOptions` is miniflare-shaped (not needed post-pivot);
`--use-wrangler-build` provides a parity oracle and escape hatch.
**Rejected:** vendoring a config parser day one (do it only if the unstable API
churns painfully); shelling out to wrangler always (slow, hides errors).

## D13 — Version identity: ULID version ids + content hashes; loader ids are `ns/name@version`

**Context:** ULIDs give ordering for UX; content hashes give idempotency/dedup;
version-scoped loader ids are the invariant that makes dynamic loading, eviction,
and multi-pod consistency safe (wdl-proven; matches CF versions model).
**Consequences:** "deploy same code twice" is a no-op; rollback is a pointer write;
no mutable loader id ever exists.

## D14 — Compat-date policy

Gate rejects uploads whose `compatibility_date` exceeds the pinned workerd's
supported date, with a clear error naming the platform's current max. Workers keep
their dates forever (upstream guarantee); platform upgrades never require user
redeploys.

## D15 — Naming

"lasso" is a placeholder codename used consistently in docs/code until the owner
picks a name. Rename is a find/replace + chart/package rename budgeted before first
public release. No Cloudflare marks in the final name.
