# The runner: supervisor + workerd + loader worker

The runner is the heart of the platform and the piece the user explicitly wanted
custom (not miniflare). A runner pod contains exactly two processes — a Go
**supervisor** (PID 1) and a stock **workerd** child — plus the platform-authored
**loader worker** running inside workerd. This mirrors Cloudflare's production
supervisor/runtime split (see research/cloudflare-production.md §1).

## 1. The supervisor (Go, `runner/supervisor`)

Responsibilities, in order of importance:

1. **Render workerd config and spawn workerd.** At startup, render the static Cap'n
   Proto **text** config from a Go template + environment (pool name, socket paths,
   gate URL, data URLs, workerd version's compat date for the loader), write it to the
   pod's tmpfs, and exec `workerd serve config.capnp --experimental
   --socket-fd http=<fd> --control-fd 3`. The listen socket is opened by the
   supervisor and passed as an FD (workerd supports `--socket-fd`), so workerd never
   binds a port itself and restarts can't drop the listener.
2. **Readiness/liveness.** Parse control-fd messages (workerd reports each socket when
   bound) → flip readiness. Liveness = supervisor's own health endpoint + a periodic
   loopback probe through workerd to the loader's `/__lasso/health` route (proves V8,
   loader, and gate connectivity are alive).
3. **Crash handling.** If workerd exits: log, backoff-respawn (the listener FD
   survives in the supervisor, so pending TCP connects queue rather than RST). Three
   rapid crashes → exit the pod nonzero and let kubelet/k8s do placement.
4. **Memory watchdog.** Sample workerd RSS (procfs). Soft threshold (default 75% of
   pod limit): set NotReady, ask gateway to drain (readiness does this implicitly),
   wait for in-flight to drain (bounded), SIGTERM workerd, respawn, Ready. Hard
   threshold is the cgroup OOM kill — same recovery path, less graceful. Loaded
   isolates are pure cache; recycling is always correct (this is the OSS substitute
   for both isolate LRU eviction and per-isolate memory caps — neither exists in stock
   workerd).
5. **Bundle cache.** Serve `GET /bundles/<version-id>` on a localhost socket to the
   loader (via a workerd `external` service): check disk cache
   (`/var/cache/lasso/bundles/`, content-verified by hash) → else fetch from gate with
   the internal token, verify, store, serve. The loader never holds gate credentials;
   the supervisor is the only code in the pod that talks to the control plane —
   exactly Cloudflare's "the sandbox cannot request any worker for which it has not
   received the appropriate key" posture, at homelab strength.
6. **Metrics & logs.** Expose Prometheus metrics (`/metrics`: workerd RSS/CPU, restart
   counts, bundle cache hits, drain events). Pipe workerd's structured-JSON
   stdout/stderr through, enriched with pool/pod labels.
7. **Version pinning.** The workerd binary is baked into the image at an exact version
   (e.g. `1.20260813.1`); the supervisor refuses to start against an unexpected
   `workerd --version` (belt-and-braces for image drift).

Explicitly **not** supervisor jobs: routing (gateway), config/UI/API (gate), storage
(data). The supervisor has no listening ports except health/metrics/bundle-localhost.

## 2. The workerd static config

One config, rendered once per process start, identical across a pool. Sketch (text
capnp; final field names to be verified against the pinned workerd's `workerd.capnp`
during M1):

```capnp
using Workerd = import "/workerd/workerd.capnp";

const config :Workerd.Config = (
  services = [
    # The trusted loader worker (embedded from the image at build time)
    ( name = "loader",
      worker = (
        modules = [ (name = "loader.js", esModule = embed "loader.js") ],
        compatibilityDate = "<pinned>",
        compatibilityFlags = ["experimental", "service_binding_extra_handlers", "enable_ctx_exports"],
        bindings = [
          ( name = "LOADER", workerLoader = ( id = "user-workers" ) ),
          ( name = "BUNDLES",  service = "supervisor-bundles" ),   # localhost bundle cache
          ( name = "DATA",     service = "lasso-data" ),           # KV/R2/D1/queues backends
          ( name = "DO_POOL",  service = "do-pool" ),              # DO routing
          ( name = "GATE_TAIL",service = "gate-tail" ),            # tail event sink
          ( name = "PUBLIC",   service = "public-internet" ),      # handed to user workers as globalOutbound
          ( name = "POOL",     fromEnvironment = "LASSO_POOL" ),
        ],
        # loader's own egress: internal mesh allowed
        globalOutbound = "internal-network",
      ) ),

    # Networks: the privilege split, expressed in capnp (wdl's trick)
    ( name = "public-internet",  network = ( allow = ["public"] ) ),
    ( name = "internal-network", network = ( allow = ["private", "public"] ) ),

    # Internal upstreams (addresses injected by supervisor via --external-addr)
    ( name = "supervisor-bundles", external = ( address = "127.0.0.1:9010" ) ),
    ( name = "lasso-data", external = ( address = "placeholder:80",
        http = ( injectRequestHeaders = [ (name = "Authorization", value = "Bearer <injected>") ] ) ) ),
    ( name = "do-pool",    external = ( address = "placeholder:80" ) ),
    ( name = "gate-tail",  external = ( address = "placeholder:80" ) ),
  ],
  sockets = [ ( name = "http", address = "*:8080", http = (
      # gateway ↔ runner hop metadata
      cfBlobHeader = "X-Lasso-Cf",
  ), service = "loader" ) ],
  logging = ( structuredLogging = true ),
);
```

Key properties:

- **User workers never appear in config.** They exist only as dynamically loaded
  isolates.
- **`globalOutbound` of every user worker is pinned to `public-internet`**, so user
  code cannot reach ClusterIP/pod ranges even though the loader can. SSRF defense in
  the runtime itself, before NetworkPolicy is even consulted.
- Internal upstream addresses are injected by the supervisor at spawn
  (`--external-addr lasso-data=…`), keeping the template environment-free.
- The DO-pool variant of this config differs only in: DO supervisor worker service
  with `durableObjectNamespaces` + `durableObjectStorage = (localDisk = "do-disk")`, a
  `disk` service on the PVC, and `preventEviction` tuning. Same image, `LASSO_MODE=do`.

## 3. The loader worker (TypeScript, `runner/loader`)

The only trusted JavaScript in the system. Compiled to a single ESM bundle at image
build (esbuild), embedded in the config. Responsibilities:

### 3.1 Request path

```
export default {
  async fetch(req, env, ctx) {
    route = parseInternalHeaders(req)          // X-Lasso-Worker: ns/name@version (set by gateway)
    if (req.url.pathname.startsWith("/__lasso/")) return handleInternal(req)  // health, dispatch
    stub = getWorker(route.id, env)            // workerLoader get-or-load
    return stub.getEntrypoint(route.entrypoint ?? "default")
               .fetch(rewriteToUserUrl(req))
  }
}
```

`getWorker(id)` on cache miss:

1. `env.BUNDLES.fetch("/bundles/" + version)` → bundle envelope (see §4) from the
   supervisor's cache. Validate magic/size.
2. Build `WorkerCode`: modules (js/wasm/text/data/json/py as declared in metadata),
   `compatibilityDate`/`flags` from metadata (never `allowExperimental` for user code),
   `env` built per §3.2, `globalOutbound: env.PUBLIC`, `tails: [ctx.exports.TailSink({props:{worker: id}})]`.
3. `env.LOADER.get(id, () => workerCode)`.
4. Record `id` in the in-isolate loaded-set; if this load *activates* a new version
   (flagged by the gateway header on first routed request), schedule eviction of
   sibling ids (`ns/name@*` ≠ this version) via the injected abort entrypoint
   (§3.4).

### 3.2 Binding materialization (the `ctx.exports` pattern)

For each binding in the version's metadata, the loader puts a **loopback entrypoint
stub with scoping props** on the user's `env`. Props are set by the loader and
invisible/unforgeable from user code:

| metadata binding | env value |
| --- | --- |
| `kv_namespace` | `ctx.exports.KV({ props: { nsId } })` — entrypoint class implementing get/put/list/delete against `env.DATA` |
| `r2_bucket` | `ctx.exports.R2({ props: { bucketId } })` |
| `d1` | `ctx.exports.D1({ props: { dbId } })` |
| `queue` (producer) | `ctx.exports.Queue({ props: { queueId } })` |
| `durable_object_namespace` | `ctx.exports.DONamespace({ props: { doNsId, className } })` — forwards to do pool |
| `service` | `ctx.exports.ServiceBinding({ props: { target: "ns/name", entrypoint } })` — re-enters the loader, so worker→worker calls stay in-process when co-resident |
| `plain_text` / `json` | value inlined |
| `secret_text` | plaintext value from the bundle envelope (decrypted by gate at envelope build; see 06-security.md) |
| `assets` | `ctx.exports.Assets({ props: { versionId } })` — serves from the bundle |

Because every privileged capability is a loopback stub, there is **no privileged
fetcher in `env` to strip** — avoiding wdl's wrapper-generation complexity entirely
(their retrospective lesson; see research/worker-loader.md §4).

These entrypoint classes are the *API-shape* layer: each implements the Workers
binding interface (`KVNamespace`, `R2Bucket`, `D1Database`, `Queue`,
`DurableObjectNamespace`) in TypeScript and translates to lasso-data's internal REST
API. Semantical fidelity notes (list ordering, metadata, conditional puts) are tracked
per-binding in [03-bindings.md](03-bindings.md).

### 3.3 Event dispatch

`/__lasso/dispatch/scheduled` and `/__lasso/dispatch/queue` (internal-only, reached
via gateway's internal path or direct service call, authenticated by the internal
token header which the gateway strips from external traffic):

```
const stub = getWorker(id, env).getEntrypoint()
const result = await stub.scheduled({ cron, scheduledTime })      // FetcherScheduledResult
const result = await stub.queue(queueName, messages)              // FetcherQueueResult
return Response.json(result)                                      // gate/data apply retry policy
```

Requires `service_binding_extra_handlers` on the loader (experimental — pinned +
conformance-tested; the flag's V8-deserializer caveat is acceptable because only the
trusted loader holds it).

### 3.4 Eviction

Every user bundle gets one injected reserved module exporting `__LassoAbort__`
(entrypoint calling `abortIsolate()` from `cloudflare:workers`) — the wdl technique.
The loader uses it to evict superseded versions; detection of "already gone" anchors
on the error behavior verified per workerd release in the conformance suite. The
injected export name is reserved: uploads defining it are rejected at gate. If
eviction proves flaky on some workerd release, the fallback is coarser but sound:
supervisor-driven process recycling on a version-churn counter.

### 3.5 Health

`/__lasso/health`: loads a canary hello-world worker via `LOADER.load()` (uncached),
invokes it, verifies data-service reachability. Used by the supervisor's readiness
probe.

## 4. Bundle envelope format (`proto/bundle.md`)

The runner-facing artifact, produced by gate, cached by supervisors:

```
{
  "format": 1,
  "id": "blog/api@01JE…9Q",
  "metadata": {                       // subset of CF multipart metadata, normalized
    "main_module": "index.js",
    "compatibility_date": "2026-08-01",
    "compatibility_flags": [...],
    "bindings": [ {type, name, ...refs} ],
    "limits": { "wallMs": 30000 }     // enforced by gateway, recorded here
  },
  "modules": [ {name, type, sha256, size, contentBase64|contentUtf8} ],
  "secrets": { "NAME": "plaintext" }  // decrypted by gate at envelope-build time
}
```

Caps enforced at gate: total modules ≤ 32 MB (headroom under workerd's 64 MB dynamic
limit), serialized env preflight ≤ 800 KB (headroom under workerd's 1 MB), module
count ≤ 512. Content-addressed: envelope body hash = the supervisor cache key,
verified on every read.

## 5. Runner image

- Base: `gcr.io/distroless/cc-debian12` (glibc ≥ 2.36 — satisfies workerd's 2.35+
  requirement) or `debian:12-slim` while debugging.
- Contents: supervisor binary, workerd binary (exact pinned version, downloaded from
  GitHub releases at build, checksum-verified), loader bundle, capnp template.
- Multi-arch: linux/amd64 + linux/arm64 (both published by upstream).
- Pod hardening (from Helm): non-root, read-only rootfs (tmpfs for config/cache),
  `RuntimeDefault` seccomp, no capabilities, optional gVisor RuntimeClass for
  dedicated pools. Detail in [06-security.md](06-security.md).

## 6. Failure modes & responses

| Failure | Response |
| --- | --- |
| workerd crash (V8 OOM, bug, poisoned isolate) | supervisor respawn, listener FD survives; repeated → pod restart. Cold-start cost only. |
| Bundle fetch fails | loader returns 503 with `X-Lasso-Error: bundle-unavailable`; gateway retries once on another pod. |
| Gate down | Existing loaded workers keep serving (bundle cache + in-memory route tables mean the data path has no synchronous gate dependency). Deploys/cron/tail pause. This must be a tested property, not an aspiration. |
| Memory creep from many versions | RSS watchdog recycle (see §1.4). |
| workerd upgrade breaks an experimental API | conformance suite blocks the image in CI; runtime never sees it. |
| Runaway user worker (CPU spin) | gateway wall-clock timeout cancels the request; supervisor CPU-share cgroup keeps the pod schedulable; per-isolate CPU metering is documented as not available (OSS limitation). |
