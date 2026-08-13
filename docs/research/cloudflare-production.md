# Research: how Cloudflare runs the Workers runtime in production

> Research date: 2026-08-13. Legend: **[PROD]** = documented production behavior
> (Cloudflare blog/docs/talks), **[OSS]** = verified behavior of open-source workerd
> (source/docs), **[INFERENCE]** = inference, labeled as such. This report is the
> template for the lasso runner/supervisor design — the closing section maps each
> production mechanism to what we imitate or skip.

## 1. The runtime process model on an edge server

### One runtime, many isolates; multiple runtime processes per machine ("cordons")

- **[PROD]** The base unit is the V8 isolate, not a process: "A single instance of the
  runtime can run hundreds or thousands of isolates, seamlessly switching between them"
  ([How Workers works](https://developers.cloudflare.com/workers/reference/how-workers-works/);
  [Cloud Computing without Containers](https://blog.cloudflare.com/cloud-computing-without-containers/)).
  Isolates start in ~5 ms and consume "an order of magnitude less memory" than a Node
  process (~3 MB vs ~35 MB).
- **[PROD]** It is **isolate-per-Worker-script**, not isolate-per-account: "Each Worker
  runs in its own V8 isolate by default"
  ([Security model docs](https://developers.cloudflare.com/workers/reference/security-model/)).
- **[PROD]** Cloudflare runs "multiple instances of the whole runtime on each machine,
  which we call 'cordons'," with Workers assigned by trust level: "a customer who signs
  up for our free plan will not be scheduled in the same process as an enterprise
  customer"
  ([Mitigating Spectre… security model](https://blog.cloudflare.com/mitigating-spectre-and-other-security-threats-the-cloudflare-workers-security-model/)).
  In the QCon talk Kenton Varda described roughly three tiers: enterprise / established
  paying / free
  ([Fine-Grained Sandboxing with V8 Isolates, QCon 2019](https://www.infoq.com/presentations/cloudflare-v8/)).
- **[PROD]** Additional processes are spawned on demand: a Worker being debugged via
  DevTools runs "in a separate process," and **dynamic process isolation** reschedules
  "any Worker with suspicious performance metrics into its own process" — detected via
  hardware performance counters measuring branch mispredictions
  ([Spectre research with TU Graz](https://blog.cloudflare.com/spectre-research-with-tu-graz/)).
  CPU-hungry Workers may also be moved out.
- **[INFERENCE]** So a machine runs: a handful of long-lived runtime processes (one per
  cordon/trust tier), a dynamic population of single-worker quarantine processes, plus
  the supervisor and front-line proxy — not process-per-tenant, and not one giant
  process.

### Supervisor / sandbox split and seccomp

- **[PROD]** The runtime (sandbox) process is deprivileged; a separate trusted
  **supervisor** does privileged work: "fetching worker code and configuration from
  disk or from other internal services" — and the sandbox "cannot request any worker
  for which it has not received the appropriate key. It cannot enumerate known workers"
  ([security blog](https://blog.cloudflare.com/mitigating-spectre-and-other-security-threats-the-cloudflare-workers-security-model/)).
- **[PROD]** The "layer 2" sandbox: Linux namespaces + seccomp, applied "after the
  process has started (but before any isolates have been loaded)", giving "a totally
  empty filesystem" and using "seccomp to block absolutely all filesystem-related
  system calls"; network access is totally prohibited — the process can communicate
  "only over local Unix domain sockets, to talk to other processes on the same system"
  (same post). The popular paraphrase "no syscalls except read/write on pre-opened FDs"
  comes from Kenton's talks; the *documented* form is the above. Treat the stricter
  phrasing as **[INFERENCE/recalled from talk]**.
- **[PROD]** TLS never enters the runtime: an inbound proxy service does "TLS
  termination (the Workers Runtime never sees TLS keys)" and forwards over Unix
  sockets; outbound `fetch()` goes through "a local proxy service" that checks the
  destination is public Internet or the zone's own origin, "not internal services".

### Scheduling, limits, eviction, cold starts

- **[PROD]** Threading (2019-era detail): "we start a thread for each incoming HTTP
  connection" and "we never run more than one isolate on a thread at a time"; Workers
  themselves are single-threaded.
- **[PROD]** Isolate eviction is LRU under a machine-level memory budget, not V8 hard
  caps: "we're going to use eight gigabytes of memory. We'll fill that up… then we'll
  evict the least recently used isolate." Kenton explicitly noted V8's built-in hard
  memory limit was avoided because it aborts the whole process
  ([QCon talk](https://www.infoq.com/presentations/cloudflare-v8/)). Docs: "do not
  store mutable state in your global scope"
  ([How Workers works](https://developers.cloudflare.com/workers/reference/how-workers-works/)).
- **[PROD]** Cold start warming during TLS handshake: "when Cloudflare receives the
  first packet during TLS negotiation, the 'ClientHello,' we hint the Workers runtime
  to eagerly load that hostname's Worker" — a ~5 ms load hidden inside the handshake
  RTT
  ([Eliminating cold starts](https://blog.cloudflare.com/eliminating-cold-starts-with-cloudflare-workers/)).
- **[PROD]** 2025 evolution — in-datacenter sharding: each Worker gets a "home" shard
  server via a consistent hash ring; the first server to receive a request proxies to
  the warm home server, adding "less than one millisecond"; overloaded shard servers
  shed load by returning "the shard client's own lazy capability" over Cap'n Proto RPC.
  Result: ~10x lower eviction/cold-start rates
  ([Eliminating Cold Starts 2: shard and conquer](https://blog.cloudflare.com/eliminating-cold-starts-2-shard-and-conquer/)).

## 2. Code/config distribution: Quicksilver

- **[PROD]** Quicksilver is Cloudflare's globally replicated KV store for
  configuration: LMDB-backed, hierarchical fan-out replication, monotonic
  sequence-numbered transaction logs, writes funneled through one elected core DC,
  ~500 ms write batching, propagation to "200 cities in 90 countries within seconds"
  ([Introducing Quicksilver](https://blog.cloudflare.com/introducing-quicksilver-configuration-distribution-at-internet-scale/)).
- **[PROD]** Worker code rides this pipeline: "It takes about three seconds between
  when you upload your code and when it's on every machine" (QCon, 2019). The runtime
  does not hold everything in memory — the supervisor lazily fetches "worker code and
  configuration from disk or from other internal services" when a request needs it.
  So: **push-replicated to local disk, pull-loaded into the runtime on demand.**
- **[PROD]** Quicksilver v2 (2025) moved away from every-server-has-everything: servers
  split into **replicas** (full dataset) and **proxies** ("persistent cache, evicting
  unused key-value pairs"), with cache misses fetched via in-DC relays and a pub/sub
  prefetch stream
  ([Quicksilver v2 pt1](https://blog.cloudflare.com/quicksilver-v2-evolution-of-a-globally-distributed-key-value-store-part-1/),
  [pt2](https://blog.cloudflare.com/quicksilver-v2-evolution-of-a-globally-distributed-key-value-store-part-2-of-2/)).
  **[INFERENCE]** With tens of millions of WFP scripts, worker code bodies are now
  effectively fetched on demand in many locations rather than pre-pushed everywhere.

## 3. Request routing to isolates

- **[PROD]** The front line: every request terminates TLS, then passes through **FL**
  ("the brain of Cloudflare"), then Pingora for cache/origin. FL1 was
  NGINX/OpenResty/LuaJIT; **FL2** is a Rust rewrite on the **Oxy** proxy framework
  ([FL2/Rust](https://blog.cloudflare.com/20-percent-internet-upgrade/);
  [Introducing Oxy](https://blog.cloudflare.com/introducing-oxy/)). The runtime sits
  behind the inbound proxy over Unix domain sockets.
- **[PROD]** Finding/creating the isolate: route lookup (zone config → worker +
  bindings, from Quicksilver) happens in FL; the runtime instance for the right cordon
  receives the request; if no isolate exists for that script, the supervisor loads code
  and the runtime creates one (~5 ms).
- **[PROD]** Per-isolate limits: "Each isolate can consume up to 128 MB of memory";
  on breach "the Workers runtime lets in-flight requests complete and creates a new
  isolate for subsequent requests." CPU: 10 ms/request free, default 30 s configurable
  to 5 min paid; wall-clock unlimited while the client is connected; exceeding limits
  surfaces as "Error 1102"
  ([Limits docs](https://developers.cloudflare.com/workers/platform/limits/)).
  Enforcement mechanism (2019 talk): a Linux timer signal that calls V8
  `TerminateExecution()`, "an uncatchable exception" — numbers have grown since, the
  mechanism presumably persists **[INFERENCE]**.
- **[PROD]** Spectre-hardening affects semantics: `Date.now()` "returns the time of the
  last I/O. It does not advance during code execution"; no concurrency primitives
  usable as timers.

## 4. Workers for Platforms and dynamic worker loading

- **[PROD]** **Dispatch namespaces** hold unlimited user Workers; user Workers run in
  "untrusted mode". A **dynamic dispatch Worker** is the entry point:
  `env.DISPATCHER.get(customerName)` returns a stub, then `userWorker.fetch(request)`
  invokes it; the dispatch Worker does auth, routing, and per-plan policy
  ([How WFP works](https://developers.cloudflare.com/cloudflare-for-platforms/workers-for-platforms/reference/how-workers-for-platforms-works/)).
- **[PROD]** **Custom limits** are a third argument to `get()`:
  `{ limits: { cpuMs: 10, subRequests: 5 } }`; on breach "the user Worker will
  immediately throw an exception"
  ([Custom limits](https://developers.cloudflare.com/cloudflare-for-platforms/workers-for-platforms/configuration/custom-limits/)).
- **[PROD]** **Tags**: up to 8 per script, used for organization and bulk CRUD. No
  public doc describes *isolate eviction* keyed by tags — treat "tag-based eviction" as
  unverified.
- **[PROD + OSS]** **Worker Loader / Dynamic Workers (2025)**: see
  research/worker-loader.md for the full OSS treatment. Production blog:
  isolates take "a few milliseconds to start… a few megabytes of memory… around 100x
  faster and 10x–100x more memory efficient than a typical container"
  ([Sandboxing AI agents, 100x faster](https://blog.cloudflare.com/dynamic-workers/)).
- **[OSS]** Dispatch namespaces are *not* in workerd — the binding union in
  workerd.capnp literally ends with `# TODO(someday): dispatch, other new features`.
  The OSS analog is `workerLoader`. Related: per-app Durable Objects for dynamic
  Workers via facets
  ([Durable Object facets blog](https://blog.cloudflare.com/durable-object-facets-dynamic-workers/)).

## 5. Limits enforcement: production vs stock workerd

- **[PROD]** 128 MB/isolate, CPU-ms metering with uncatchable termination, subrequest
  counting. Runaway-worker defense is layered: CPU timer signals per request, LRU
  memory eviction at the process level, dynamic process isolation for
  suspicious/CPU-hungry workers, and cordons.
- **[OSS]** Stock workerd enforces **none of this**. Verified:
  `class NullIsolateLimitEnforcer final: public IsolateLimitEnforcer` with empty
  methods is the only implementation used by the server
  ([server.c++](https://github.com/cloudflare/workerd/blob/main/src/workerd/server/server.c++);
  raised in [workerd issue #49](https://github.com/cloudflare/workerd/issues/49)).
  There is no per-worker limit config in workerd.capnp. README: "workerd is not a
  hardened sandbox… you must run it inside an appropriate secure sandbox, such as a
  virtual machine."
- **[OSS]** What you do get: a **process-global** `v8Flags` config option — so
  `--max-old-space-size`-style heap caps apply to the whole process, not per isolate.
- **[INFERENCE]** Therefore the only faithful way to approximate production limits with
  stock workerd is **process-per-tenant (or per small tenant group)**: v8Flags heap cap
  + cgroup v2 `memory.max`/`cpu.max` + wall-clock request timeouts in the supervisor.
  That's exactly the architecture Cloudflare itself falls back to for
  untrusted/suspicious workers (dynamic process isolation), applied universally at
  small scale. Kenton's stated reason for avoiding V8 hard heap limits — hitting one
  aborts the process — becomes a *non-problem* when the process contains only one
  tenant.

## 6. Control plane: uploads → active at the edge

- **[PROD]** Upload via API creates an immutable **version**; a **deployment** maps
  traffic to one or two versions ("route a percentage of requests to the new
  version"), with per-request probabilistic splitting, optional **version affinity**
  to pin clients, and version overrides for subrequests
  ([Versions & deployments](https://developers.cloudflare.com/workers/configuration/versions-and-deployments/),
  [Gradual deployments](https://developers.cloudflare.com/workers/configuration/versions-and-deployments/gradual-deployments/)).
- **[PROD]** Distribution is Quicksilver: config changes globally "within seconds".
- **[PROD]** Secrets: "encrypted text values" attached as bindings; write-only after
  creation; delivered to code as ordinary env bindings
  ([Secrets docs](https://developers.cloudflare.com/workers/configuration/secrets/)).
  Internal storage/KMS details are not published **[gap]**.
- **[PROD]** Cloudflare-internal config changes now go through **Health Mediated
  Deployments** — progressive rollout with health metrics and automatic rollback
  ([Scaling with safety](https://blog.cloudflare.com/safe-change-at-any-scale/)).

## 7. Runtime crash/restart/health; rolling out runtime versions

- **[PROD]** V8 security patches trigger a fully automated build-and-release: "patch
  gap now under 24 hours".
- **[PROD/OSS]** Compatibility dates make daily/weekly runtime replacement safe:
  runtime updates "should never cause a Worker that is already deployed to stop
  functioning"; "old dates will continue to be supported… forever"
  ([Compatibility dates](https://developers.cloudflare.com/workers/configuration/compatibility-dates/)).
- **[PROD]** Zero-drop restarts for edge proxies (Oxy family): systemd-owned listening
  sockets survive process replacement; new process takes all new connections
  immediately, old process drains with an upper bound; coordinated by their `shellflip`
  Rust crate
  ([Oxy: the journey of graceful restarts](https://blog.cloudflare.com/oxy-the-journey-of-graceful-restarts/)).
  **[INFERENCE]** The Workers runtime restarts the same way behind its Unix sockets;
  isolates are stateless-by-contract, so a runtime process crash just means cold
  starts, and the supervisor respawns it.

## What a reasonable homelab clone looks like

Target shape: Kubernetes + stock workerd + a custom Go supervisor. Honest summary:
**Cloudflare's security model = isolates + things you cannot have (proprietary
limiters, perf-counter Spectre detection, fleet scale). Their fallback for untrusted
code = process isolation + seccomp + a privileged supervisor — and that part you can
copy almost verbatim, and it's cheap at homelab scale.**

### Worth imitating

1. **The supervisor/sandbox split (copy this exactly).** The supervisor holds all
   control-plane credentials, fetches code/config, and hands the runtime only what it
   asked for by opaque ID. Speak to workerd over a Unix socket or localhost HTTP; give
   it capability-only config (workerd's config model is already designed for this —
   bindings, no ambient network, `globalOutbound` per worker).
2. **Process-per-tenant (or per trust tier) instead of shared cordons.** This is the
   substitute for the missing `IsolateLimitEnforcer`: one workerd process per trust
   domain in its own container/cgroup with `memory.max` (e.g., 192–256 MB per tenant
   tier to emulate the 128 MB heap + runtime overhead), `cpu.max`, `pids.max`, and
   process-wide `v8Flags` heap caps as a second line. OOM-kill = Cloudflare's "toss the
   isolate, make a new one," at process granularity. At homelab tenant counts the
   density argument for shared processes doesn't apply, and you get Spectre isolation
   for free.
3. **The layer-2 sandbox.** Empty read-only rootfs, no capabilities, seccomp
   (`RuntimeDefault` or stricter), NetworkPolicy denying all egress except the
   supervisor's egress path. Route all `fetch()` through `globalOutbound` to a
   validated egress service (their local-proxy SSRF defense).
4. **Emulate dispatch namespaces with the OSS `workerLoader` binding.** One stock
   "loader" worker per runtime process with a `workerLoader` binding; its callback
   pulls code from the control plane; deploys don't require workerd restarts. Enforce
   `cpuMs`-like budgets as wall-clock timeouts in the supervisor/gateway (true CPU
   metering isn't available in OSS).
5. **Compatibility dates + immutable versions in the control plane.** Store every
   upload as an immutable version with its compat date; never mutate. This is what
   lets you upgrade workerd continuously without breaking tenants — the single
   cheapest high-value idea in the whole architecture.
6. **Zero-drop restarts via socket handover.** workerd supports socket-activation-style
   FD inheritance (`--socket-fd`); alternatively keep listening sockets in the
   supervisor and treat workerd processes as disposable backends (drain old, spawn
   new). In K8s: rolling pods with `maxUnavailable: 0` gets most of this for free.
7. **Pull-through code cache.** Control-plane DB + blob store as source of truth;
   supervisor caches bundles on local disk and loads lazily on first request, with a
   simple change feed (SSE/long-poll) instead of Quicksilver. Seconds-level propagation
   is trivial at this scale.

### Skip (scale-driven, not architecture-driven)

- **Quicksilver itself** — polling + pub/sub solves the homelab version.
- **TLS-handshake (SNI) pre-warming** — cold starts on one warm machine are
  single-digit ms.
- **The shard/consistent-hash-ring isolate placement** — with 2–3 nodes, a sticky
  hostname→pod hash in the gateway gets 95% of it.
- **Dynamic process isolation with perf-counter Spectre detection** — stronger
  isolation comes statically from process-per-trust-domain.
- **Per-request gradual deployments / version affinity** — a per-worker "pin version"
  field plus blue-green at the process level is enough (keep the schema open for it).
- **Trust-level cordons** — subsumed by process-per-trust-domain.
- **True CPU-ms enforcement and 128 MB per-isolate caps inside one shared workerd** —
  not achievable in OSS; don't fight it, change the granularity to the process.

## Key sources

[Mitigating Spectre… security model](https://blog.cloudflare.com/mitigating-spectre-and-other-security-threats-the-cloudflare-workers-security-model/) ·
[Security model docs](https://developers.cloudflare.com/workers/reference/security-model/) ·
[QCon: Fine-Grained Sandboxing with V8 Isolates](https://www.infoq.com/presentations/cloudflare-v8/) ·
[Cloud Computing without Containers](https://blog.cloudflare.com/cloud-computing-without-containers/) ·
[How Workers works](https://developers.cloudflare.com/workers/reference/how-workers-works/) ·
[Eliminating cold starts](https://blog.cloudflare.com/eliminating-cold-starts-with-cloudflare-workers/) ·
[Shard and conquer](https://blog.cloudflare.com/eliminating-cold-starts-2-shard-and-conquer/) ·
[Introducing Quicksilver](https://blog.cloudflare.com/introducing-quicksilver-configuration-distribution-at-internet-scale/) ·
[Quicksilver v2 pt1](https://blog.cloudflare.com/quicksilver-v2-evolution-of-a-globally-distributed-key-value-store-part-1/) /
[pt2](https://blog.cloudflare.com/quicksilver-v2-evolution-of-a-globally-distributed-key-value-store-part-2-of-2/) ·
[FL2/Rust](https://blog.cloudflare.com/20-percent-internet-upgrade/) ·
[Oxy graceful restarts](https://blog.cloudflare.com/oxy-the-journey-of-graceful-restarts/) ·
[TU Graz dynamic process isolation](https://blog.cloudflare.com/spectre-research-with-tu-graz/) ·
[Introducing workerd](https://blog.cloudflare.com/workerd-open-source-workers-runtime/) ·
[Limits](https://developers.cloudflare.com/workers/platform/limits/) ·
[WFP how it works](https://developers.cloudflare.com/cloudflare-for-platforms/workers-for-platforms/reference/how-workers-for-platforms-works/) ·
[Dynamic Workers](https://blog.cloudflare.com/dynamic-workers/) ·
[Worker Loader docs](https://developers.cloudflare.com/workers/runtime-apis/bindings/worker-loader/) ·
[DO facets](https://blog.cloudflare.com/durable-object-facets-dynamic-workers/) ·
[Gradual deployments](https://developers.cloudflare.com/workers/configuration/versions-and-deployments/gradual-deployments/) ·
[Compatibility dates](https://developers.cloudflare.com/workers/configuration/compatibility-dates/) ·
[Secrets](https://developers.cloudflare.com/workers/configuration/secrets/)
