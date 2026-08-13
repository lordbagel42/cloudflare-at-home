# Research: workerd ground truth

> Research date: 2026-08-13. Primary sources: the workerd GitHub repo (README,
> `src/workerd/server/workerd.capnp` config schema, `src/workerd/server/workerd.c++`
> CLI, `src/workerd/server/server.c++`, `src/workerd/io/compatibility-date.capnp`,
> `src/workerd/util/autogate.h`), the Cloudflare announcement blog post, the npm
> registry, and the GitHub releases page. Schema/CLI facts were read directly from
> current `main` source.

## 1. What workerd is

workerd (lowercase, "-d" for daemon) is the open-source (Apache 2.0) JavaScript/Wasm
server runtime that is the core of Cloudflare Workers. Cloudflare positions it for three
uses: self-hosting Workers-designed apps ("application server"), local development (it
is the engine inside Miniflare 3 / `wrangler dev`), and as a programmable HTTP proxy
(forward or reverse) — the config format alone can express a TLS-terminating reverse
proxy with no JS.

Architecture facts:

- **V8 isolates, one process.** Many Workers run in a single workerd process, each in
  its own V8 isolate with its own global scope. All built-in APIs are native C++
  (KJ/jsg), so isolates share one copy of the runtime code — per-worker overhead is
  small.
- **Nanoservices.** Workers can call each other via *service bindings*; the callee runs
  in the same process/thread, so cross-service calls have "performance of a local
  function call" — zero network hop, no serialization of request bodies through the
  kernel.
- **Event-driven, server-first.** Workers never open listen sockets; the runtime pushes
  events (HTTP requests, queue events, alarms) to them. All ingress is declared in
  config as `sockets` routed to named `services`.
- **Capability-based bindings.** A Worker starts with no privileged resources;
  everything is granted through explicit named bindings in config (this is also the
  SSRF story — global `fetch()` is itself just a service, default-restricted to
  publicly-routable addresses).
- **Homogeneous deployment model.** The intended cluster shape is "every machine runs
  every service" — you deploy the whole config to every node and load-balance across
  identical nodes, rather than scheduling services individually.

**What's included vs. production.** The announcement blog post is explicit: workerd is
"just the runtime"; the production service adds "additional security, deployment
mechanisms, orchestration, and so much more." Concretely, from source:

- Included: the full JS/Wasm engine and Workers API surface (fetch/Request/Response,
  WebCrypto, streams, WebSockets, Node.js compat layer, RPC, TCP `connect()`), Durable
  Objects (SQLite-backed locally), Cache API plumbing, queues *bindings*, KV/R2
  *binding shims*, D1 *client shim*, tail workers, Python workers (Pyodide), Wasm,
  alarms via DO.
- NOT included: any distributed storage backend. `kvNamespace`, `r2Bucket`, and `queue`
  bindings are protocol adapters that convert API calls into HTTP requests to a service
  you name in config (see §2). There is no KV store, no R2 object store, no D1 server,
  no queue broker, no cron trigger scheduler, no deployment/upload API, no multi-tenant
  orchestration, and no distributed DO routing shipped in workerd. Miniflare supplies
  local "simulators" for these on top of workerd; a self-hosted platform must do the
  same.
- Cloudflare also warns it is "not independent": it's developed primarily for
  Cloudflare's own service; internal priorities win over external contributions.

Sources: https://github.com/cloudflare/workerd (README),
https://blog.cloudflare.com/workerd-open-source-workers-runtime/

## 2. Configuration: the Cap'n Proto format

Config is a Cap'n Proto file — either text `.capnp` that imports
`/workerd/workerd.capnp` and defines a `const config :Workerd.Config`, or a **binary**
Cap'n Proto message (`workerd serve --binary`), which is what higher-level tooling like
wrangler/miniflare generates. Full reference is the commented schema:
https://github.com/cloudflare/workerd/blob/main/src/workerd/server/workerd.capnp

Top level `Config`: `services`, `sockets`, `v8Flags` (raw V8 flags, "use at your own
risk"), `extensions`, `autogates`, and `logging :LoggingOptions` (`structuredLogging`
bool = JSON logs; `stdoutPrefix`/`stderrPrefix`; the older top-level
`structuredLogging` field is deprecated in favor of this).

**Services** (named union):

- `worker` — see below.
- `network` — fetch-to-network with CIDR `allow`/`deny` lists plus specials `"public"`,
  `"private"`, `"local"`, `"network"`, `"unix"`, `"unix-abstract"`.
- `external` (ExternalServer) — forward everything to one address over http/https/tcp,
  with TLS options, `certificateHost`, header injection.
- `disk` (DiskDirectory) — bare-bones HTTP GET/PUT onto a directory; `writable`,
  `allowDotfiles`; no Content-Type guessing; JSON directory listings.

If you don't define a service named `internet`, one is implicitly defined allowing
`["public"]` only; it backs global `fetch()` unless a worker overrides `globalOutbound`.

**Sockets**: `name`, `address` (`*:8080`, IPv4/6, `unix:/path`, `unix-abstract:name`,
DNS names), protocol union `http`/`https(+tlsOptions)`/`tcp(+tlsOptions)`, and `service`
(a `ServiceDesignator`: service name + optional named `entrypoint` + optional `props`
JSON delivered as `ctx.props`). `HttpOptions` covers proxy-style requests (full-URL
request lines, so a Worker can be an HTTP forward proxy), `forwardedProtoHeader`,
`cfBlobHeader` (how `request.cf` is encoded/parsed across hops), request/response header
injection, and `capnpConnectHost` (CONNECT upgrades to Cap'n Proto RPC, used for JSRPC
between workerd instances).

**Worker**: code as union of `modules` (first module = main) | `serviceWorkerScript` |
`inherit` (clone another worker with overridden bindings — same isolate, different
`env`; derived workers must set their own `durableObjectStorage` and
`durableObjectUniqueKeyModifier`). Module types: `esModule`, `commonJsModule`
(+`namedExports`), `text`, `data`, `wasm`, `json`, `pythonModule` (two slots obsolete:
nodeJsCompatModule, pythonRequirement). `compatibilityDate` is required (workerd's own
version *is* the max supported date; a newer date is rejected), `compatibilityFlags`
optional. `moduleFallback` names a fallback service consulted for unresolved module
specifiers (used by dev tooling; see the `FallbackServiceRequest` struct).
`tails`/`streamingTails` route tail events to tail-worker services.

**Bindings** (`Worker.Binding` union) — each has a `name`, surfaces on `env` (modules)
or globals (service-worker syntax):

- `text`, `data`, `json` — plain values (secrets are just `text`/`json` bindings; there
  is no distinct "secret" type in workerd).
- `fromEnvironment` — value read from a system environment variable via `getenv()`.
- `cryptoKey` — pre-imported WebCrypto key (raw/hex/base64/pkcs8/spki/jwk) with
  `extractable = false` support, so key material can be made unleakable to JS. Useful
  for platform secret handling.
- `wasmModule` (deprecated; use wasm modules), `service` (service binding, i.e.
  Fetcher, with entrypoint + props), `durableObjectNamespace`, `durableObjectClass`
  (class-only, for facets), `parameter` (abstract binding filled by an inheriting
  worker).
- `kvNamespace :ServiceDesignator` — "Requests to the namespace will be converted into
  HTTP requests targeting the given service name." I.e., the KV API is a client shim;
  the storage is any HTTP service you point it at (another Worker, an `external`
  server, or a `disk` service fronted by a Worker). Same pattern for `r2Bucket` and
  `queue` (producer side).
- `wrapped :WrappedBinding` — wraps inner bindings with a JS module (`moduleName`,
  `entrypoint`, `innerBindings`). This is how D1, AI, Vectorize, Workflows, etc. are
  exposed: workerd ships built-in `cloudflare-internal:` modules
  (`src/cloudflare/internal/`: `d1-api.ts`, `ai-api.ts`, `vectorize-api.ts`,
  `workflows-api.ts`, `images-api.ts`, ...) that translate the typed API into HTTP
  calls on an inner `service`/fetcher binding. **There is no `d1Database` binding type
  in the schema — D1 locally = wrapped binding around an HTTP service that speaks the
  D1 query protocol** (Miniflare implements that service with SQLite).
- `hyperdrive` (designator + database/user/password/scheme), `analyticsEngine` (events
  forwarded as HTTP to a service; "subject to change and requires the `--experimental`
  flag"), `unsafeEval` (enables eval/new Function), `memoryCache` (in-process shared
  cache with `maxKeys`/`maxValueSize`/`maxTotalValueSize` limits), `workerLoader`
  (dynamically load Workers from code at runtime, with optional shared cache `id` —
  central to this project; see research/worker-loader.md), `workerdDebugPort` (dev-only
  RPC into another workerd's debug port).

Also per-worker: `globalOutbound` (redirect global fetch to any service — a platform
can force all egress through a filtering proxy worker), `cacheApiOutbound` (where
`caches.default`/`caches.open()` requests go — without it the Cache API is a no-op
passthrough locally; Miniflare wires it to a SQLite/blob-backed cache service. The
no-op-by-default claim is high-confidence but inferred from behavior/docs, not from a
schema comment), `containerEngine` (`localDocker` with socket path — DO-attached
containers via local Docker, dev/test only), and local-dev Cloudflare Access emulation
(`accessBlobHeader`, `accessBindingService`).

**Extensions** (top-level): lists of JS modules (`name` must be a URL like
`my-ext:module` under the new module registry; `internal = true` hides from user code)
that can define user-importable APIs and back `wrapped` bindings. See
`samples/extensions/`.

**Experimental bits**: the `--experimental` CLI flag gates (a) compat flags annotated
`$experimental` in `compatibility-date.capnp` (e.g. `python_workers_development`,
streaming tail workers, the catch-all `experimental` flag), (b)
`durableObjectStorage.localDisk` ("EXPERIMENTAL; SUBJECT TO BACKWARDS-INCOMPATIBLE
CHANGE"), (c) `analyticsEngine` bindings, and others. Expect no stability guarantees
behind it.

Minimal example (README): a service `main` wrapping a worker with
`serviceWorkerScript = embed "hello.js"`, `compatibilityDate`, plus socket
`( name = "http", address = "*:8080", http = (), service = "main" )`. A fuller real
example incl. Durable Objects is `samples/durable-objects-chat/config.capnp`.

## 3. Distribution, platforms, sandboxing

- **npm**: package `workerd` (bin wrapper, `engines: node >= 16`) with
  platform-specific optional deps: `@cloudflare/workerd-linux-64`,
  `workerd-linux-arm64`, `workerd-darwin-64`, `workerd-darwin-arm64`,
  `workerd-windows-64`. Latest at research time: `1.20260813.1`. Prebuilt Linux
  binaries require **glibc 2.35+** (Ubuntu 22.04-era; plain Alpine/musl won't run
  them), macOS 13.5+, x86-64 with SSE4.2/CLMUL or arm64 with CRC extensions.
- **GitHub releases** carry the same binaries as release assets.
- **Docker: there is NO official Cloudflare-published workerd Docker image** (Docker
  Hub has only community images, e.g. `jacoblincool/workerd`,
  Cyb3r-Jak3/docker-workerd). The repo contains Dockerfiles for building/dev, not a
  published runtime image. For Kubernetes you build your own image (typically copy the
  npm binary onto a glibc base). The one image Cloudflare does publish in this
  ecosystem is `cloudflare/proxy-everything` (the container-egress interceptor sidecar
  referenced in `DockerConfiguration`).
- **Building from source**: Bazel; a large C++/V8 build. Avoid; consume binaries.
- **Sandboxing / untrusted code — Cloudflare's explicit position.** README: "workerd is
  not a hardened sandbox"; it tries to isolate workers but "workerd on its own does not
  contain suitable defense-in-depth against the possibility of implementation bugs.
  When using workerd to run possibly-malicious code, you must run it inside an
  appropriate secure sandbox, such as a virtual machine." The blog post adds it cannot
  on its own defend against Spectre-class hardware attacks or V8 zero-days, and that
  production Workers layers many additional protections (process sandboxing/seccomp,
  isolate scheduling defenses) that are *not* in the open-source release. Note
  `docs/hardening.md` in the repo is about internal C++ coding hardening (KJ macros,
  pointer discipline), not deployment sandboxing — there is no official seccomp profile
  shipped. Practical implication: treat one workerd process = one tenant trust domain,
  and wrap it in gVisor/Kata/Firecracker or at least strict
  seccomp+non-root+read-only-fs if tenants are mutually untrusted. Security reports go
  through Cloudflare's HackerOne bug bounty.

## 4. Runtime operations

Subcommands (from `workerd.c++`): `serve`, `compile` (produce a self-contained binary
embedding config + all embedded source; the resulting binary reuses the serve options),
`test` (runs `*.wd-test` unit tests; has `--no-verbose`, `--predictable`,
`--gc-stress`, `--all-autogates`, `--compat-date`, `--config-only`), `pyodide-lock`,
`make-pyodide-baseline-snapshot`, and (fuzz builds) `fuzzilli`.

`serve` options (ground truth from source):

- Config parsing: `-I/--import-path`, `-b/--binary` (binary capnp config), positional
  `<config-file> [<const-name>]`.
- Overrides: `-s/--socket-addr <name>=<addr>`, `-S/--socket-fd <name>=<fd>` (inherit
  pre-opened FDs — good for socket activation), `-d/--directory-path <name>=<path>`
  (override DiskDirectory paths), `-e/--external-addr <name>=<addr>` (override
  ExternalServer addresses). These overrides are the intended way to vary environment
  without editing config.
- `--control-fd <fd>`: emits control messages (currently: reports the port each socket
  bound, when ready) — this is how Miniflare/wrangler detect readiness; useful for a
  k8s readiness probe integration in a supervisor.
- `-i/--inspector-addr <addr>`: V8 inspector (Chrome DevTools protocol) listener for
  debugging/profiling. `--debug-port <addr>`: privileged RPC that "allows access to all
  services in the process. For use by miniflare and local development only."
- `-w/--watch`: watch config files and the binary, reload on change — "not recommended
  in production." **Otherwise config is loaded once at startup; there is no
  hot-reload/admin API.** Dynamic behavior must come from `workerLoader` bindings, the
  module fallback service, or restarting the process.
- `--experimental`: unlocks experimental features (see §2).
- Python: `--pyodide-package-disk-cache-dir`, `--pyodide-bundle-disk-cache-dir`,
  `--python-save-snapshot`, `--python-load-snapshot`, `--python-snapshot-dir`.

**Logging/observability**: logs go to stdout/stderr; `logging.structuredLogging = true`
switches to JSON (caveat: not for old-registry service-worker-syntax workers' logs).
There is **no built-in Prometheus/metrics endpoint** — `reportMetrics(IsolateObserver&)`
hooks exist in C++ but the OSS server wires them to observers that don't export
anywhere; Cloudflare's exporters are internal. Options: tail workers (`tails` config —
receive structured trace/log/exception events per request in a Worker you control, and
forward them wherever), structured stdout logs, the inspector protocol (CPU/heap
profiling), and Perfetto tracing in custom builds.

**Resource limits: effectively none in OSS workerd.** `server.c++` instantiates
`NullIsolateLimitEnforcer` — literally commented "IsolateLimitEnforcer that enforces no
limits" — for every isolate. Production's 128 MB heap / CPU-ms limits are enforced by
internal implementations of these interfaces that are not shipped. Local knobs you do
have: `v8Flags` (process-wide, at your own risk), OS/cgroup limits on the whole process
(k8s pod limits), and structuring tenants across processes. Plan on **per-pod
(per-process) limits, not per-isolate limits**.

## 5. Durable Objects locally

- Every DO namespace needs `className` + either `uniqueKey` (stable secret string;
  object IDs are cryptographically derived from it — "DO NOT LOSE this key") or
  `ephemeralLocal` (no storage API at all; arbitrary string IDs).
- Storage backends (`durableObjectStorage`): `none` | `inMemory` (persists for the
  process lifetime only; "intended for local testing") | `localDisk =
  "<DiskDirectory service name>"` — experimental. On localDisk, workerd creates
  `<dir>/<uniqueKey>/<id>.sqlite` (+ `-wal`, `-shm`) — **one SQLite database file per
  object**. On k8s that directory must be a PV; DO data is pinned to whichever workerd
  instance owns the disk.
- workerd uses SQLite to back *all* DOs; `enableSql = true` per namespace exposes the
  `storage.sql` API (hidden by default to emulate non-SQLite-backed production
  namespaces).
- `preventEviction` disables the default lifecycle (evicted after ~10 s inactivity,
  expire 70 s after clients disconnect) — a workerd-only knob.
- DO alarms work against the SQLite storage in local dev (well-established via
  Miniflare; the alarm persistence code path on `localDisk` was not independently
  re-verified — low uncertainty).
- Containers-attached DOs: `container = (imageName = ...)` per namespace + worker-level
  `containerEngine.localDocker` (Docker socket path + egress interceptor image).
  Explicitly "local development and testing purposes."
- **Key limitation, stated in the schema itself**: "TODO(someday): Support distributing
  objects across a cluster. At present, objects are always local to one instance of the
  runtime." DO uniqueness is per-process; two workerd processes will each happily
  instantiate "the same" object. Any multi-node DO story (consistent routing of object
  IDs to one process) is ours to build at the routing layer.

## 6. Autogates, compatibility dates, releases

- **Compatibility dates/flags**: identical system to production
  (`compatibility-date.capnp`): each behavior change has an enable-flag, a
  disable-flag, and usually a default-on date. "Updating workerd to a newer version
  will never break your JavaScript code." Flags marked `$experimental` require
  `--experimental`.
- **Versioning**: `v<major>.<YYYYMMDD>.<patch>` (e.g. `1.20260813.1`); the date *is*
  the maximum compatibility date that build supports — configs requesting a newer
  compat date won't run. Releases are effectively **daily-to-weekly**, published to
  GitHub releases and npm in lockstep (npm version = tag minus the `v`).
- **Autogates** (`autogates` config list / `src/workerd/util/autogate.h`): temporary
  kill-switches for risky internal rollouts, distinct from compat flags — "not intended
  … for long-term feature gating … An autogate never becomes a compatibility flag."
  Names expand to `workerd-autogate-<kebab-case>`. For a self-hosted platform these can
  generally be ignored (defaults track Cloudflare's stable path).

## 7. Python and other languages

- **Python Workers** are in-tree: module type `pythonModule`; bundles are "converted
  into a JS/WASM Worker Bundle prior to execution" via **Pyodide** (CPython on Wasm).
  Compat flags: `python_workers` (schema comment still warns Python Workers are
  experimental and subject to change) and experimental `python_workers_development`.
  Operationally important for k8s: workerd **fetches Pyodide bundles and Python
  packages from the internet at runtime** unless you pre-populate
  `--pyodide-package-disk-cache-dir` / `--pyodide-bundle-disk-cache-dir`; cold-start
  mitigation via memory snapshots. Samples: `samples/pyodide*`,
  `samples/repl-server-python`.
- **Wasm**: first-class (`wasm` modules → `WebAssembly.Module`), so
  Rust/C/Go-compiled-to-Wasm works as in production.
- **Other languages** compile to JS/Wasm; there is no other native language runtime.
- Rich Node.js compatibility via the `nodejs_compat` flag family.

## 8. Clustering / multi-node

Confirmed: **workerd has no clustering, no multi-node coordination, no shared state,
and no distributed DO routing.** Evidence: the schema TODO quoted in §5; the blog's
homogeneous-deployment model explicitly delegates distribution to "a load balancer in
front of the fleet"; no config surface exists for peer discovery. The only
cross-process primitives are ordinary HTTP (`external` services), the Cap'n Proto
`capnpConnectHost` mechanism for JSRPC between processes, and the dev-only
`--debug-port`/`workerdDebugPort`. Multi-node = N processes behind load balancing, with
DO stickiness and all storage backends (KV/R2/D1/queues/cache) implemented by the
platform as HTTP services the bindings point at.

## Key sources

- Repo/README: https://github.com/cloudflare/workerd
- Config schema (authoritative reference):
  https://github.com/cloudflare/workerd/blob/main/src/workerd/server/workerd.capnp
- CLI: `src/workerd/server/workerd.c++`; server internals (NullIsolateLimitEnforcer):
  `src/workerd/server/server.c++`
- Compat flags: `src/workerd/io/compatibility-date.capnp`; autogates:
  `src/workerd/util/autogate.h`
- Built-in Cloudflare API shims: https://github.com/cloudflare/workerd/tree/main/src/cloudflare/internal
- Samples: https://github.com/cloudflare/workerd/tree/main/samples (esp.
  `durable-objects-chat/config.capnp`)
- Announcement: https://blog.cloudflare.com/workerd-open-source-workers-runtime/
- Releases: https://github.com/cloudflare/workerd/releases ; npm:
  https://www.npmjs.com/package/workerd

Flagged uncertainties: (a) Cache API being a no-op without `cacheApiOutbound` is
inferred, not schema-documented; (b) DO alarm persistence on `localDisk` was not
independently re-verified; (c) queue *consumer* event delivery mechanics were traced
separately — see research/worker-loader.md for `Fetcher.queue()`.
