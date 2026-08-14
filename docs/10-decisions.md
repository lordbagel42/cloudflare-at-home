# Decision record

ADR-style. Each entry: decision, context, alternatives rejected, consequences.
Status of all: **accepted** (2026-08-13) unless noted. D2/D4/D5 revised and D16 added
2026-08-14 (owner direction: no `workerLoader` as the core mechanism; mirror
Cloudflare's internal architecture).

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

## D2 (revised 2026-08-14) — Process-per-worker-version with static config; no `workerLoader`

Each active worker version runs as its own workerd process with a statically rendered
Cap'n Proto config, spawned lazily and reaped when idle by the pod supervisor. Deploys
are per-worker blue-green process swaps. `workerLoader` is not used.
**Context:** owner direction — avoid building the platform on `workerLoader`; stay
architecturally close to what Cloudflare actually runs. Cloudflare's runtime loads
code into isolates via a proprietary supervisor channel that OSS workerd does not
expose; the faithful OSS translation of "isolate created lazily, evicted under
pressure, hard-limited" is the *process*: Cloudflare themselves run workers as
separate processes when they need enforceable limits (dynamic process isolation), and
their self-hosting guidance is static config + process replacement. Bonus: static
config unlocks workerd's **native binding shims and native Durable Objects**, which
dynamic workers cannot use, and removes the biggest experimental-API dependency.
**Rejected:**
- *`workerLoader` loader-worker design* (the 2026-08-13 draft; preserved in
  research/worker-loader.md and git history): best deploy latency and density, but
  builds the platform's core on an explicitly experimental API, gives no per-worker
  memory/CPU containment, makes the platform own isolate eviction, and blocks native
  DO/binding support. Remains the documented fallback if process density ever becomes
  the binding constraint.
- *One shared workerd per pool with all workers in one config, full-process swap per
  deploy*: couples every worker's availability (drops all pool WebSockets on any
  deploy), no per-worker limits, config grows monotonically.
**Consequences:** per-worker memory overhead (one workerd process each — measured in
M0 spike; mitigated by lazy spawn + idle reaping); cold start = process spawn
(tens of ms, honest and measured) instead of isolate load (single-digit ms);
supervisor gains a process-manager role (spawn/route/drain/reap); the remaining
experimental surface shrinks to `service_binding_extra_handlers` (cron/queue dispatch
into the harness worker) and DO `localDisk` storage.

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

## D4 (revised 2026-08-14) — Native workerd binding shims pointed at lasso-data

Per-worker config declares real `kvNamespace`, `r2Bucket`, `queue`, wrapped-D1,
`cacheApiOutbound`, and `durableObjectNamespace` bindings, each pointed at an
`external` service entry for lasso-data with per-binding scoping headers
(namespace/bucket/db id + auth) injected via `injectRequestHeaders`. lasso-data
implements the server side of workerd's internal binding protocols (miniflare's
simulators are the reference implementation, kept current by Cloudflare).
**Context:** D2's pivot to config-declared workers makes native shims available —
maximum API fidelity because the *client side is workerd's own C++*, exactly what
production runs. The previous `ctx.exports` stub design existed only because dynamic
workers can't use config bindings.
**Rejected:** TS stub classes over loopback RPC (previous design — now unnecessary
indirection and a fidelity risk); vendoring miniflare simulator workers (ties us to
`miniflare:shared` internals).
**Consequences:** lasso-data must track workerd's internal wire protocols, which are
*not* covered by compat-date guarantees → the conformance suite gates workerd
upgrades on binding-protocol behavior too; scoping/auth lives in config-injected
headers the user code never sees; secrets become plain `text` bindings rendered into
per-process config (tmpfs, mode 0400).

## D16 — Runner process model: supervisor as in-pod process manager

The supervisor (PID 1) owns a dynamic set of child workerd processes, one per
resident worker version, each listening on its own unix domain socket; the supervisor
is the pod's only TCP listener and routes `X-Lasso-Worker` to the matching child,
spawning on miss (bundle fetch → modules to tmpfs → render config → spawn → control-fd
ready) and reaping on idle TTL, version supersession (drain-then-reap), crash-looping,
or memory pressure.
**Context:** this reproduces Cloudflare's supervisor↔runtime relationship — the
privileged side fetches code and manages lifecycle; the deprivileged runtime only
executes — with the isolate lifecycle mapped onto processes (lazy create, LRU evict,
limit-kill-recreate).
**Rejected:** pod-per-worker via the k8s API (puts etcd/scheduler on the deploy path —
violates D1's latency goal; reserved for `isolation: dedicated` namespaces at *pool*
granularity); SO_REUSEPORT sharing of one TCP port among children (loses routing
control and per-version drain precision).
**Consequences:** the supervisor grows a routing/proxy component (Go, unix-socket
reverse proxy — small and testable); per-worker `v8Flags` heap caps become possible
(default 192 MB, configurable per worker); worker crash blast radius is exactly one
worker.

## D5 (revised 2026-08-14) — Trust domain = worker process; pools are resource domains

**Context:** with D2/D16, every worker already has its own process, so inter-worker
isolation no longer leans on the V8 isolate boundary at all — a strict upgrade over
both earlier designs, and stronger than Cloudflare's own shared cordons. OSS workerd
still isn't a hardened sandbox, so the boundary that matters is the OS process +
container.
**Rejected:** pretending isolate boundaries are a security guarantee; requiring
gVisor/VMs by default (homelab cost).
**Consequences:** pools now exist for *resource* grouping (cgroup budgets, gVisor
option, scheduling), not code isolation; `isolation: dedicated` namespaces get their
own pool for cgroup-level guarantees and blast-radius control; per-request CPU
metering remains unavailable (documented) — wall-clock timeout at the gateway plus
per-process heap caps are the backstops.

## D6 — Go for services, TypeScript for Workers-ecosystem code

**Context:** Go: single-binary services, first-class k8s clients, easy static
distroless images. TS: the harness *must* be a worker; the CLI lives in the
Workers/wrangler ecosystem.
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

## D13 — Version identity: ULID version ids + content hashes; process identity is `ns/name@version`

**Context:** ULIDs give ordering for UX; content hashes give idempotency/dedup;
version-scoped runtime identity (originally loader ids, now process identities) is
the invariant that makes deploys, drains, and multi-pod consistency safe (matches
CF's versions model).
**Consequences:** "deploy same code twice" is a no-op; rollback is a pointer write;
a running process never changes version — replacement only.

## D14 — Compat-date policy

Gate rejects uploads whose `compatibility_date` exceeds the pinned workerd's
supported date, with a clear error naming the platform's current max. Workers keep
their dates forever (upstream guarantee); platform upgrades never require user
redeploys.

## D15 — Naming

"lasso" is a placeholder codename used consistently in docs/code until the owner
picks a name. Rename is a find/replace + chart/package rename budgeted before first
public release. No Cloudflare marks in the final name.
