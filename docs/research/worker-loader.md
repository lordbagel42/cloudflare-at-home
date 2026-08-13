# Research: dynamic worker loading (`workerLoader`) in workerd

> Research date: 2026-08-13. Sources: workerd `main` branch source (verbatim quotes from
> files cited by path), Cloudflare Dynamic Workers docs, Cloudflare blog posts, and the
> wdl repo (github.com/wdl-dev/wdl). This mechanism is the heart of the lasso runner —
> read this before touching the runner or loader worker.

## 1. The `workerLoader` binding and the JS API

### 1.1 Schema (verbatim)

From `src/workerd/server/workerd.capnp` (lines ~446–460):

```capnp
workerLoader :group {
  # A binding representing the ability to dynamically load Workers from code presented at
  # runtime.
  #
  # A Worker loader is not just a function that loads a Worker, but also serves as a
  # cache of Workers, automatically unloading Workers that are not in use. To that end, each
  # Worker must have a name, and if a Worker with that name already exists, it'll be reused.

  id @27 :Text;
  # Optional: The identifier associated with this Worker loader. Multiple Workers can bind to
  # the same ID in order to access the same loader, so that if they request the same name
  # from it, they'll end up sharing the same loaded Worker.
  #
  # (If omitted, the binding will not share a cache with any other binding.)
}
```

**Caution:** "automatically unloading Workers that are not in use" describes the
*intended contract* (and production behavior), not what OSS workerd does today — see
§6. wdl's code comments state flatly: "workerd's loader never evicts on its own."

The optional `id` names a shared cache namespace: in `src/workerd/server/server.c++`
(~line 5983), loaders with a config `id` go into `workerLoaderNamespaces` (a `HashMap`
keyed by id, shared across all workers binding that id); loaders without an id are
private per binding.

### 1.2 Exact JS API

From `src/workerd/api/worker-loader.h`:

- `env.LOADER.get(id, getCodeCallback)` → returns a `WorkerStub` **synchronously**.
  Signature: `get(name: string, getCode: () => Promise<WorkerCode>)`. The callback is
  invoked only on cache miss.
- `env.LOADER.load(code)` — "Shortcut for `get(null, () => code)`" — creates a fresh,
  unnamed (uncached) worker each call. Docs: "Use `load()` when the code is always new,
  such as for one-off AI-generated tool calls."
- `WorkerStub.getEntrypoint(name?, { props?, limits? })` → `Fetcher`
  (service-binding-like stub; fetch + JSRPC). `"default"` and omitted both mean the
  default export.
- `WorkerStub.getDurableObjectClass(name?, { props?, limits? })` → `DurableObjectClass`
  — "The only use of this type is to pass to `ctx.facets.get()`"
  (`src/workerd/api/actor.h` ~line 418).

`WorkerStub` header comment: "This is not a stub for a specific entrypoint, but instead
the entire Worker, allowing the caller to call any entrypoint (and specify arbitrary
props)."

### 1.3 `WorkerCode` — exact fields (struct `WorkerCode` in `worker-loader.h`)

| Field | Type / semantics |
|---|---|
| `compatibilityDate` | required string |
| `compatibilityFlags` | optional `string[]` |
| `allowExperimental` | optional bool, default false — required to use experimental compat flags in the child; **only allowed if the calling worker itself has the `experimental` compat flag** ("'allowExperimental' is only allowed when the calling worker has the 'experimental' compat flag set."). Also: "a worker loader being trusted with specific experimental flags should not imply that it can delegate that trust to its dynamic workers." |
| `limits` | optional `{ cpuMs?: uint32, subRequests?: uint32 }` (`src/workerd/io/io-channels.h` line 514) — parsed but NOT enforced in OSS, see §2.5 |
| `mainModule` | required string, key into `modules` |
| `modules` | `Record<string, string \| Module>`; plain string ⇒ ES module if name ends `.js`, Python module if `.py`; anything else rejected (`.ts` gets a special error telling you to bundle with `@cloudflare/worker-bundler`). Object form: **exactly one** of `js`, `cjs`, `text`, `data` (ArrayBuffer), `json`, `py`, `wasm` (compiled wasm bytes). So esm/cjs/wasm/text/data/json/python are all supported. |
| `env` | optional object — "Any RPC-serializable value!" See §1.4. |
| `globalOutbound` | optional `Fetcher \| null`. Verbatim: "If omitted, inherit the current worker's global outbound. If `null`, block the global outbound (all requests throw errors)." A blocked outbound throws: "This worker is not permitted to access the internet via global functions like fetch(). It must use capabilities (such as bindings in 'env') to talk to the outside world." |
| `tails` | optional `Fetcher[]` — tail workers **are** supported on dynamic workers. |
| `streamingTails` | optional `Fetcher[]` — requires `allowExperimental: true`. |

Size caps hard-coded in `worker-loader.c++`: total module bodies ≤ **64 MB**
(`MAX_DYNAMIC_WORKER_CODE_SIZE`, "mirrors the documented paid Worker uncompressed size
limit"), serialized `env` ≤ **1 MB** (`MAX_DYNAMIC_WORKER_ENV_SIZE`).

### 1.4 What `env` can carry

In `server.c++` (`WorkerStubImpl::start`, ~line 5240), env is a `Frankenvalue` whose
capability table is rewritten into the child's I/O channel table. Exactly three
capability kinds are accepted:

1. **`SubrequestChannel`** — service bindings / Fetchers **passed by reference**
   (including entrypoint stubs of other dynamic workers, `ctx.exports` loopback stubs,
   and DO *stubs* — DO stubs serialize as plain Fetchers, dropping `id`/`name` props).
2. **`ActorClassChannel`** — `DurableObjectClass` objects (so a dynamic worker can
   receive a DO class in env and pass it to its own `ctx.facets.get()`).
3. **`RpcChannel`** — RPC stubs.

Anything else structured-clonable is passed by value. Non-serializable objects throw:
"Dynamic 'env' contains one or more objects that are not supported for use in 'env',
although they would be supported in 'props'."

### 1.5 Durable Objects inside dynamic workers

**A dynamically loaded worker cannot declare its own Durable Object namespaces.**
Ground truth: `WorkerStubImpl::start` sets `.localActorConfigs = EMPTY_ACTOR_CONFIGS`
and the dynamic `WorkerDef` has no actor namespace channels (`server.c++` ~5273). There
is no `durableObjectNamespaces`/storage config in `WorkerCode`.

The supported pattern is **facets**: the parent extracts the class via
`worker.getDurableObjectClass("App")` and instantiates it as a facet of a
config-declared ("supervisor") Durable Object via `this.ctx.facets.get(name, callback)`
(`ctx.facets` in `src/workerd/api/actor-state.h` line 726, non-experimental). Docs
(developers.cloudflare.com/dynamic-workers/usage/durable-object-facets/): "The
supervisor's database and the facet's database are stored together as part of the same
overall Durable Object. The dynamic code cannot read the supervisor's database — it
only has access to its own." Facet lifecycle: `get()` / `abort()` (stops, keeps
storage) / `delete()` (destroys storage). Storage is whatever the supervisor DO's
namespace uses — in workerd `durableObjectStorage` = `inMemory` or `localDisk` (the
latter still marked "** EXPERIMENTAL; SUBJECT TO BACKWARDS-INCOMPATIBLE CHANGE **").
There is also a `durableObjectClass` config binding type for wiring classes statically.

### 1.6 `id` semantics, caching, eviction/replacement

- Caching is per loader-namespace, keyed by the string id
  (`WorkerLoaderNamespace::isolates` HashMap). `findOrCreate` ⇒ same id = same isolate;
  the callback is not called again.
- Docs warn the contract is *at most* caching, never identity: "there is no guarantee:
  a later call with the same ID may instead start a new isolate from scratch" and
  "ensure that the callback always returns exactly the same content, when called for
  the same ID". The `load()` implementation comment confirms the production runtime
  "can actually evict the isolate while a stub still exists, as long as there is no
  active request on the stub, and then recreate the isolate on the next request."
- **In OSS workerd there is no automatic eviction** (see §6). Two ways to
  replace/evict:
  1. **New id per version** (immutable content-addressed/versioned ids) — the pattern
     both Cloudflare's design and wdl assume.
  2. **Explicit abort from inside**: `import { abortIsolate } from
     "cloudflare:workers"` — `EntrypointsModule::abortIsolate`
     (`src/workerd/api/workers-module.h` line 91, unconditional `JSG_METHOD`, no
     experimental gate). For a dynamic worker, `IoContext::abortIsolate()` triggers the
     `abortIsolateCallback` which "Removes the isolate from the isolates map" so
     "subsequent loadIsolate() calls with the same name will create a fresh isolate".
     For a **non-dynamic** worker it aborts the whole process. Note the upstream TODO:
     "Should abort all outstanding calls to the isolate causing them to throw" —
     currently admitted calls drain.

## 2. Dynamic vs config-declared workers — limitations

1. **Tails**: yes — `WorkerCode.tails` (the dynamic worker *sends* tail events to
   fetchers you supply).
2. **Cron/queues**: dynamic workers cannot appear in workerd's `sockets` or be
   cron/queue targets in config. Instead you invoke handlers directly on the stub:
   `fetcher.scheduled(options)` and `fetcher.queue(queueName, messages, metadata?)`
   **exist on Fetcher** — `src/workerd/api/http.h` lines 367–425 — but only when the
   **calling** worker has compat flag `service_binding_extra_handlers`, which is itself
   `$experimental` (`src/workerd/io/compatibility-date.capnp` line 295): "Allows
   service bindings to call additional event handler methods on the target Worker…
   WARNING: this flag exposes the V8 deserialiser to users via `Fetcher#queue()`
   `serializedBody`. Historically, this has required a trusted environment to be safe."
   So this works, but only under `--experimental` and only in a trusted host worker
   (exactly how wdl uses it).
3. **Entrypoints/RPC**: fully supported — `getEntrypoint(name)` reaches any exported
   `WorkerEntrypoint`, with JSRPC methods, plus per-stub `props`.
4. **Nesting**: **not possible in OSS workerd.** A dynamic worker's only bindings come
   from `env`; a `WorkerLoader` object has no serializer and isn't one of the three
   accepted cap types, and the dynamic `WorkerDef` never populates
   `workerLoaderChannels`. Workaround: pass the child a loopback RPC stub that loads
   workers on its behalf in the parent.
5. **Limits/memory (OSS)**: workerd uses `NullIsolateLimitEnforcer` — verbatim:
   "IsolateLimitEnforcer that enforces no limits." (`server.c++` line 3231). Moreover
   the per-entrypoint `limits` passed to `getEntrypoint()` are silently **dropped** in
   workerd (`getEntrypointResolved` constructs its channel without them, ~5202–5210),
   and `DynamicWorkerSource.limits` isn't wired into the dynamic `WorkerDef`. The
   Dynamic Workers limits docs ("cpuMs / subRequests… immediately throw an exception")
   describe **Cloudflare production**, not OSS workerd. **In a self-hosted platform you
   get no per-tenant CPU or heap enforcement from workerd itself.**
6. **`moduleFallback`** is forced to `kj::none` for dynamic workers — dynamic workers
   must be fully self-contained module bundles.
7. **Python**: supported (`py` modules; auto-adds `python_workers` and
   `disable_python_external_sdk` flags), with docs warning "Python Workers are
   currently much slower to start than JavaScript Workers."

## 3. Adjacent mechanisms: `ctx.exports`, `props`, facets

- **`ctx.exports` (loopback bindings)**: gated by compat flag `enable_ctx_exports`,
  default-on for compatibility date ≥ **2025-11-17**.
  `ctx.exports.<ExportedClass>({ props })` creates a stub of your own worker's named
  entrypoint with chosen props — "each call is an RPC back to the original object in
  your loader Worker". This is the canonical way to hand a dynamic worker
  capability-scoped platform bindings: "you need to bind the resource to your loader
  Worker and create a custom binding that wraps it"
  (developers.cloudflare.com/dynamic-workers/usage/bindings/).
- **`props`**: set at stub creation (`getEntrypoint(name, {props})` or
  `ctx.exports.Class({props})`), read by the entrypoint as `this.ctx.props`. The
  *callee entrypoint* sees them; the code that received the stub can't inspect or
  change them — this is what makes them safe tenant/namespace scoping tokens.
- **Dispatching non-fetch events into loaded workers**:
  `stub.getEntrypoint().scheduled()/.queue()` return `{outcome, noRetry…}`-style
  results (`FetcherScheduledResult`/`FetcherQueueResult`). wdl's `runtime/dispatch.js`
  does exactly `await entry.scheduled(scheduled)` and checks
  `scheduledResult.outcome === "exception"`.

## 4. How wdl uses `workerLoader` in practice

Repo: https://github.com/wdl-dev/wdl — "a self-hosted multi-tenant Workers platform
with multi-replica failover, built on stock Cloudflare workerd… loads immutable Worker
versions dynamically from Redis/Valkey through workerd's `workerLoader` API". Pins
workerd npm 1.20260811.1 and runs `workerd serve … --experimental`.

**Topology**: `gateway` (workerd, public :8080) resolves `{ns, worker, version}` from
Redis route state, strips client-supplied internal headers, and forwards to the runtime
pool with `x-worker-id: <ns>:<worker>:<version>` + `x-worker-prefix`. Two isolated
runtime pools (`user-runtime`, `system-runtime`) run the same loader worker under
different capnp configs; the privilege split is expressed **in capnp networking**, not
k8s policy: tenant workers' `globalOutbound` is pinned to a `public-network` service
(`allow = ["public"]`) so the internal mesh is TCP-unreachable, while the loader itself
keeps `internal-network`.

**Code feed**: on cache miss, the loader callback fetches a bounded envelope
(`WDLLOAD!` magic, content type `application/vnd.wdl.runtime-load`) from a colocated
**redis-proxy sidecar** (Rust) which reads immutable bundle keys from Valkey and
decrypts envelope-encrypted secrets at cold load. wasm/data modules get standalone byte
copies "so tenant-visible buffers cannot expose adjacent modules or secrets."

**Version pinning / eviction**: "Workers are loaded by immutable id
`<ns>:<worker>:<version>`. Promotion creates a new id, so active-version changes
naturally cold-load a fresh isolate." Since "workerLoader cache has no LRU," wdl keeps
a per-process `Map` of loaded ids and, on active-version cold loads, **evicts sibling
historical versions** by calling an injected reserved entrypoint:
`stub.getEntrypoint("__WdlAbort__").abort("wdl-evict")`, whose implementation is just
`abortIsolate(reason)` from `cloudflare:workers`. Success detection is anchored on the
error-message prefix `"internal error; reference ="` — with the comment that "workerd
upgrades must reverify that behavior." Service-binding loads pin frozen versions and
deliberately don't evict siblings.

**Bindings wiring**: entirely the `ctx.exports` + `props` pattern. The loader exports
entrypoint classes `KV`, `Assets`, `QueueProducer`, `D1Database`, `R2Bucket`,
`ServiceBinding`, `DurableObjectNamespace`, `InternalAuthBackend`; `buildWorkerEnv`
materializes tenant env as e.g. `ctx.exports.KV({ props: { ns, id } })`. KV/queues go
to redis-proxy; R2 is runtime-signed S3; D1/DO/Workflows go through hidden backend
Fetchers that a generated wrapper module strips from tenant-visible `env` before user
code runs. Every load attaches a tail worker via `workerCode.tails`; outbound is
`globalOutbound: env.PUBLIC_NETWORK`. Control pre-validates the serialized env against
a headroomed estimate of workerd's 1 MB env budget at deploy time so oversized envs
fail in the control plane, not at cold load.

**Durable Objects**: run in a separate `do-runtime` service using **native workerd
facets + SQLite (`localDisk`) storage**, fronted by owner leases; D1 likewise
per-database owner leases. Failover: "owner records carry task identity, lease expiry,
and a monotonic generation fence, so stale replicas fail closed after a takeover."

**Honest assessment — copy vs avoid:**

- *Copy*: immutable id-per-version keys (makes eviction mostly unnecessary and
  multi-replica-safe); `ctx.exports` + props for capability-scoped bindings;
  capnp-level network privilege separation for tenant outbound; env-size preflight in
  the control plane; attaching a platform tail worker via `WorkerCode.tails`; treating
  scheduled/queue dispatch as result envelopes over `stub.scheduled()/queue()`.
- *Copy with caution*: the `__Abort__`/`abortIsolate()` sibling-eviction trick — it's
  the only in-band eviction mechanism in stock workerd, but it depends on an injected
  reserved entrypoint, an undocumented error-string anchor, and un-guaranteed upstream
  behavior (wdl itself flags this).
- *Avoid / be aware*: no LRU means **the supervisor owns memory pressure** — wdl's
  answer is bounded per-pool tenancy plus process replacement, not isolate accounting.
  The module-rewriting/wrapper-generation layer (hiding privileged fetchers from tenant
  env, prototype-pollution defenses) is substantial complexity you inherit the moment
  you put privileged fetchers in `env` instead of using pure loopback stubs — prefer
  keeping *all* privileged channels behind `ctx.exports` stubs so there's nothing to
  strip. Their KV `list()` is HSCAN-backed (unordered) — an example of
  API-shape-compatible-but-semantically-different tradeoffs to document, not hide.

## 5. The module fallback service (`moduleFallback`)

`Worker.moduleFallback @13 :Text` — the value is the HTTP address of a fallback
service. Authoritative comment in `src/workerd/server/fallback-service.h`:

> "The fallback service is a mechanism used **only in workerd local development**. It
> is used to use an external http service to resolve module specifiers dynamically if
> the module is not found in the static bundles. A worker must be configured to use the
> fallback service and workerd must be started with the **--experimental** CLI flag."

Protocol: V1 = GET with query-string specifier/referrer; V2 = POST with JSON body,
URL-typed specifier/referrer, import attributes included. The service returns a JSON
module-config, a 301 redirect to another specifier, or an error; resolution order is
bundle → builtin → fallback, with a redirect cache. It's how
wrangler/vitest-pool-workers serve node_modules on demand. **Verdict: dev-only by
explicit upstream declaration**, and it does not apply to dynamically loaded workers at
all (`moduleFallback = kj::none` for dynamic `WorkerDef`s). Don't build a production
platform on it; feed complete module sets through `WorkerCode.modules`.

## 6. Long-running workerd: isolate/memory behavior, experimental status

**Isolate lifecycle in OSS workerd** (from `server.c++`
`WorkerLoaderNamespace`/`WorkerStubImpl`):

- **Named (`get(id, …)`)**: entries live in the namespace's `isolates` map and are
  removed *only* by `abortIsolate()` from inside the worker (or server shutdown).
  There is **no idle timer, no LRU, no memory-pressure eviction**. A long-running
  process loading many distinct ids grows monotonically unless you abort old ones or
  recycle the process. After an abort removes the map entry, outstanding stubs keep the
  old isolate alive until they drain/GC.
- **Unnamed (`load()`)**: no map entry; the isolate lives as long as the JS
  `WorkerStub` (plus a startup-completion ref). When the stub is GC'd, the
  `WorkerService`/isolate is torn down — so workerd does free isolates for dropped
  unnamed dynamic workers, but only via GC of your stub references.
- Isolates are *not* reused across different ids; each loaded worker gets its own
  isolate (named `<namespace>:<id>`).
- **No heap/CPU enforcement** in OSS; `hasExcessivelyExceededHeapLimit` exists in the
  interface (`src/workerd/io/limit-enforcer.h`) but the OSS enforcer enforces nothing.

**`--experimental` status (as of 2026-08-13, workerd main): still required.**
`server.c++` (~line 4946):

> "Worker loader bindings are an experimental feature which may change or go away in
> the future. You must run workerd with `--experimental` to use this feature."

`service_binding_extra_handlers` is likewise still `$experimental`. Meanwhile the
*hosted* product: Worker Loader API announced Sept 2025 (Code Mode post, closed beta);
**open beta since 2026-03-24** for all paid plans
(developers.cloudflare.com/changelog/post/2026-03-24-dynamic-workers-open-beta/,
blog.cloudflare.com/dynamic-workers/). Docs live at
developers.cloudflare.com/dynamic-workers/. So: production-side it's an open beta;
OSS-side the binding remains flag-gated experimental with an explicit compat-break
warning. **Consequence for lasso: pin the workerd version exactly and run a conformance
suite on every upgrade** (see the roadmap).

**Uncertainties flagged by the researcher:**

- Whether Cloudflare production enforces `WorkerCode.limits`/entrypoint `limits` is
  docs-asserted; in OSS both are parsed and then ignored — verified in source but worth
  an empirical test if the API shape matters.
- The no-nesting and env-capability-whitelist conclusions were verified by absence of
  handling in source plus runtime error messages, not by executing tests.
- The `abortIsolate` drain-vs-cancel behavior carries an explicit upstream TODO and may
  change across workerd releases (wdl re-verifies it per upgrade; so should we).
