# The runner: supervisor + per-worker workerd processes

> Revised 2026-08-14 for the process-per-worker model (no `workerLoader`; see D2/D16).
> Ground truth for everything here: research/static-config-verification.md.

A runner pod contains one Go **supervisor** (PID 1) and a dynamic set of stock
**workerd child processes — one per resident worker version** — each with a small
platform **harness worker** co-declared in its config. This mirrors Cloudflare's
production supervisor/runtime split, with their isolate lifecycle (lazy create, LRU
evict, limit-kill-recreate) mapped onto processes.

## 1. The supervisor (Go, `runner/supervisor`)

The supervisor is the pod's only TCP listener and the only holder of platform
credentials. Its jobs:

### 1.1 Routing (the in-pod front line)

Accept HTTP on `:8080`, read `X-Lasso-Worker: ns/name@version` (set by
lasso-gateway; the supervisor rejects requests without it or with an unknown
internal-auth header on `/__lasso/*` paths), and reverse-proxy to the matching child
process's unix domain socket (`/run/lasso/<ns>_<name>_<version>.sock`). Miss →
spawn-on-demand (§1.2), then proxy. The proxy is a plain Go `httputil.ReverseProxy`
over unix sockets with streaming/WebSocket passthrough — no buffering, no rewriting
beyond internal-header hygiene.

### 1.2 Process lifecycle

**Spawn** (cold start, budget ≤ 300 ms warm-cache / ≤ 1 s with bundle fetch):

1. Resolve version → bundle: disk cache (`/var/cache/lasso/bundles/<version>`,
   hash-verified) or fetch from gate `/internal/bundles/<version>` with the internal
   token. Bundles with secrets are cached in tmpfs only.
2. Materialize modules to a per-process tmpfs dir; render `config.capnp` (text
   template, §2) with mode 0400.
3. Spawn `workerd serve config.capnp --experimental --control-fd 3` with: dedicated
   unix socket path, per-worker `v8Flags` heap cap (default
   `--max-old-space-size=192`), env stripped to essentials.
4. Wait for the control-fd readiness event (workerd emits
   `{"event":"listen","socket":…}` per socket when actually ready — verified
   behavior, same signal miniflare uses). Mark routable.

Measured baseline: ~32 ms process start + eager compile for a trivial worker,
~41 MB RSS (all config-declared workers compile eagerly at startup — fine, since
each process holds exactly one user worker + the tiny harness).

**Swap (deploy)** — two modes, chosen by whether the version declares DO namespaces:

- *Stateless workers (blue-green overlap)*: spawn `@new`, wait ready, flip the
  routing map atomically, SIGTERM `@old`. workerd's SIGTERM handling drains
  in-flight requests (including WebSockets) and exits when empty — verified; there
  is **no built-in drain timeout**, so the supervisor SIGKILLs after a grace period
  (default 30 s; WebSocket-heavy workers configurable up to 10 min).
- *DO workers (drain-then-start)*: two live processes must never share a DO storage
  directory (verified: no cross-process arbitration; split-brain otherwise). So:
  unroute → SIGTERM old → wait for exit (or SIGKILL at grace) → spawn new → route.
  Brief per-worker unavailability, documented; requests during the window get 503 +
  `Retry-After: 1` and the gateway retries once.

**Reap**: LRU idle eviction (default TTL 15 m, DO pool default: no reaping) and
version supersession. **Respawn**: on child exit, exponential backoff; 3 rapid
crashes → mark the worker failing (503s with `X-Lasso-Error: worker-crashlooping`,
reported to gate for visibility) without affecting any other worker.

**Memory**: per-process heap caps make a leaking worker abort alone (V8 OOM aborts
only its own process — the reason Cloudflare avoids per-isolate hard caps
disappears at process granularity). The pod-level RSS watchdog reaps the
largest-RSS idle process first under pressure; the pod cgroup is the backstop.

### 1.3 Everything else

- **Readiness/liveness**: pod-ready = supervisor healthy + route feed connected (or
  snapshot loaded); per-worker health is per-process and never fails the pod.
- **Prewarm**: route-feed deploy events for locally-resident workers trigger
  immediate swap; a `prewarm` hint on first-route lets the gateway ask pods to spawn
  before traffic lands.
- **Dispatch**: `/__lasso/dispatch/{scheduled,queue}` (internal-token-authed)
  proxied to the target process's harness worker.
- **Metrics/logs**: per-worker-process RSS/CPU from procfs
  (`lasso_worker_memory_bytes{worker,version}`…), spawn/reap/respawn counters,
  cold-start histograms; children's structured-JSON stdout/stderr enriched with
  worker/version labels and forwarded to pod stdout.
- **Version pinning**: workerd binary baked into the image; supervisor refuses to
  start on `workerd --version` mismatch.

Explicitly not supervisor jobs: hostname routing (gateway), storage (data),
scheduling (gate).

## 2. The per-worker workerd config

Rendered from a Go text template per (worker, version). Sketch — final field names
verified against the pinned `workerd.capnp` in the M0 spike:

```capnp
using Workerd = import "/workerd/workerd.capnp";

const config :Workerd.Config = (
  services = [
    # ---- the user worker ----
    ( name = "user",
      worker = (
        modules = [ /* rendered from bundle metadata: esModule/wasm/text/data/json/py */ ],
        compatibilityDate = "<from metadata>",
        compatibilityFlags = [ /* from metadata; experimental flags rejected at gate */ ],
        bindings = [
          # plain values and secrets, inlined at render time (config is 0400 tmpfs):
          ( name = "ENV_NAME", text = "value" ),
          # native shims, one external service per binding (scoped by headers, §3):
          ( name = "CACHE_KV",  kvNamespace = "kv-CACHE_KV" ),
          ( name = "MEDIA",     r2Bucket    = "r2-MEDIA" ),
          ( name = "DB",        wrapped = ( moduleName = "cloudflare-internal:d1-api",
                                            innerBindings = [ (name = "fetcher", service = "d1-DB") ] ) ),
          ( name = "JOBS",      queue = "q-JOBS" ),
          ( name = "OTHER_API", service = "svc-OTHER_API" ),   # service binding → supervisor route
          # DO namespaces (only on the do pool):
          ( name = "ROOMS", durableObjectNamespace = "Room" ),
        ],
        durableObjectNamespaces = [ ( className = "Room",
                                      uniqueKey = "<stable platform key>",
                                      enableSql = true ) ],
        durableObjectStorage = ( localDisk = "do-disk" ),      # DO pool only; no --experimental needed
        globalOutbound = "public-internet",
        cacheApiOutbound = "cache-default",                    # v1.5; omitted in v1 = honest no-op
        tails = [ "harness" ],
      ) ),

    # ---- the harness worker (platform JS, embedded from image) ----
    ( name = "harness",
      worker = (
        modules = [ (name = "harness.js", esModule = embed "harness.js") ],
        compatibilityDate = "<pinned>",
        compatibilityFlags = ["service_binding_extra_handlers"],   # the one experimental flag
        bindings = [
          ( name = "USER", service = "user" ),        # for fetcher.scheduled()/queue()
          ( name = "TAIL_SINK", service = "gate-tail" ),
          ( name = "ASSETS_DIR", service = "assets-disk" ),
        ],
        globalOutbound = "null-network",              # harness needs no internet
      ) ),

    # ---- networks & upstreams ----
    ( name = "public-internet", network = ( allow = ["public"] ) ),
    ( name = "null-network",    network = ( allow = [] ) ),
    ( name = "assets-disk", disk = ( path = "<bundle assets dir>" ) ),
    ( name = "do-disk",     disk = ( path = "/data/do/<worker>", writable = true ) ),
    # one entry per binding, address + scoping headers injected at render:
    ( name = "kv-CACHE_KV", external = ( address = "<lasso-data host:port>",
        http = ( injectRequestHeaders = [
          ( name = "Authorization",       value = "Bearer <internal token>" ),
          ( name = "X-Lasso-KV-Namespace", value = "<nsId>" ) ] ) ) ),
    # r2-MEDIA / d1-DB / q-JOBS analogous, each with its own scoping header
    ( name = "svc-OTHER_API", external = ( address = "<supervisor http addr>",
        http = ( injectRequestHeaders = [
          ( name = "X-Lasso-Worker", value = "<target ns/name@pinned-active>" ),
          ( name = "X-Lasso-Internal", value = "<token>" ) ] ) ) ),
    ( name = "gate-tail", external = ( address = "<gate internal addr>", http = (
        injectRequestHeaders = [ ( name = "Authorization", value = "Bearer <token>" ),
                                 ( name = "X-Lasso-Worker", value = "<ns/name@version>" ) ] ) ) ),
  ],
  sockets = [ ( name = "http", address = "unix:/run/lasso/<proc>.sock",
                http = ( cfBlobHeader = "X-Lasso-Cf" ), service = "user" ) ],
  logging = ( structuredLogging = true ),
);
```

Properties worth naming:

- **The user worker's bindings are workerd's own native client shims** — the same
  C++ that talks to production Cloudflare backends talks to lasso-data. Wire
  protocols are specified in research/static-config-verification.md §1 and
  conformance-tested per workerd release.
- **Capability scoping lives in `injectRequestHeaders`** on per-binding `external`
  services: user code can never see, alter, or forge the namespace/auth headers, and
  holds no credentials — only capabilities. `globalOutbound = public-internet`
  blocks the internal mesh at the runtime layer, before NetworkPolicy.
- **Secrets are `text` bindings** rendered into the 0400 tmpfs config of a process
  that belongs to that worker alone. (CF delivers secrets as env bindings the same
  way.)
- **Service bindings** loop through the supervisor with a pinned target version
  header, so worker→worker calls follow the same routing, draining, and metrics
  path as external traffic (and hop pods only when the target isn't resident).
- The socket routes **directly to the user worker** — zero platform JS on the hot
  request path. The harness is reached only via `tails` (workerd pushes trace
  events) and via the supervisor's dispatch proxy.

## 3. The harness worker (TypeScript, `runner/harness`)

The only platform JS in a worker process; embedded at image build; zero runtime npm
dependencies. Three duties:

1. **Event dispatch**: `fetch()` handler accepts
   `/dispatch/scheduled {cron, scheduledTime}` and
   `/dispatch/queue {queueName, messages[]}` (reachable only through the
   supervisor), calls `env.USER.scheduled(...)` / `env.USER.queue(...)` — the
   `service_binding_extra_handlers` methods — and returns the outcome envelope
   (`FetcherScheduledResult`/`FetcherQueueResult`) as JSON for gate/lasso-data to
   apply retry policy. Miniflare's broker/scheduled workers are the reference
   implementations of both call shapes.
2. **Tail forwarding**: implements the tail worker interface; batches trace items
   (logs, exceptions, outcomes) and POSTs to gate via `TAIL_SINK` with worker
   identity injected by config headers. Sampling caps applied here.
3. **Asset serving** (v1): exposes `/assets/<path>` reading from the bundle's asset
   directory via the `ASSETS_DIR` disk service; the user worker reaches it through
   an `ASSETS` service binding when the bundle declares assets.

The harness holds the process's single experimental compat flag; the user worker
never gets experimental flags (gate rejects them at upload).

## 4. Bundle format (`proto/bundle.md`)

Unchanged from the previous design except envelopes are consumed by the
*supervisor* (not in-runtime code):

```
{
  "format": 2,
  "id": "blog/api@01JE…9Q",
  "metadata": { main_module, compatibility_date, compatibility_flags,
                bindings: [...], durable_objects: {...}, crons: [...],
                limits: { wallMs } },
  "modules":  [ {name, type, sha256, size} ],   # contents as sibling blobs
  "secrets":  { "NAME": "plaintext" }           # present only in gate→supervisor transfer
}
```

Caps at gate: modules ≤ 64 MB total (paid-plan parity; no dynamic-loading 1 MB env
constraint anymore), module count ≤ 512, config-render preflight (binding refs
resolvable, reserved service names `user|harness|*-disk` unclaimable via binding
names — the renderer prefixes all binding-derived service names).

## 5. Runner image

- Base `gcr.io/distroless/cc-debian12` (glibc ≥ 2.36 satisfies workerd's 2.35+
  floor); `debian:12-slim` variant for debugging.
- Contents: supervisor, pinned workerd binary (GitHub releases, checksum-verified at
  build), harness bundle, config template. Multi-arch amd64/arm64.
- Pod hardening per 06-security.md (non-root, read-only rootfs + tmpfs, seccomp
  RuntimeDefault, no caps, optional gVisor per pool).

## 6. Failure modes

| Failure | Response |
| --- | --- |
| Worker process crash (V8 OOM/abort, bug) | Supervisor respawn with backoff; blast radius = that worker; crash-loop → failing state, others unaffected. |
| Bundle fetch fails | 503 `X-Lasso-Error: bundle-unavailable`; gateway retries once on another pod. |
| Gate down | Resident processes keep serving (routing map + bundle cache are local); spawns of *cached* workers still work; only uncached cold spawns, deploys, cron, tail pause. Tested property. |
| Hung request blocking drain | Grace-period SIGKILL (workerd has no drain timeout of its own — verified). |
| Pod memory pressure | Watchdog reaps largest idle process; cgroup OOM as backstop chooses within the pod, respawn recovers. |
| workerd upgrade breaks a wire protocol or the dispatch flag | Conformance suite blocks the image in CI (protocols are internal contracts, not compat-date-covered). |
| Runaway CPU | Gateway wall-clock timeout cancels the request; cgroup CPU shares keep the pod schedulable; per-request CPU metering documented as unavailable. |
