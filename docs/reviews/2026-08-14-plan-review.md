# Plan review — 2026-08-14

> Scope: full review of docs/00–10 + research/, focused on gaps, under-explained
> sections, and internal contradictions, with special attention to **deploying
> Durable Objects safely and scalably** (the owner's stated worry). External ground
> truth was researched for this review: how Cloudflare runs DO deploys/storage
> internally, and Fly.io's engineering (LiteFS, Litestream, Corrosion, fly-replay,
> volume/deploy semantics) as prior art. Sources cited inline.
>
> Verdict up front: the architecture is sound and the research under it is unusually
> solid. The pivot to process-per-worker is the right call and matches both
> Cloudflare's own fallback posture and what Fly charges money for. But the plan has
> **four correctness-level gaps** (DO migrations, alarms-vs-lazy-spawn, version
> routing authority, the internal-token capability hole), a **DO durability/
> availability story that is thinner than the rest of the design**, and a set of
> smaller under-specifications listed at the end. All are fixable in docs before M0.

---

## 1. Critical findings (fix in docs before implementation starts)

### C1. The platform-wide internal token inside per-worker configs contradicts the capability model

02-runner.md §2 renders, into every worker's config:

```capnp
( name = "kv-CACHE_KV", external = ( ... injectRequestHeaders = [
    ( name = "Authorization", value = "Bearer <internal token>" ), ... ) ) )
```

and 04-control-plane.md says **one chart-generated internal bearer token** is used by
"all internal services". Meanwhile 06-security.md claims "worker processes receive
capabilities, never credentials" and cites Cloudflare's rule that the sandbox "cannot
request any worker for which it has not received the appropriate key."

These cannot all be true. The rendered config **is** readable by the workerd process
that runs untrusted code (that's how injectRequestHeaders works — workerd holds the
plaintext). The design's own threat model (Black Hat 2026 memory-corruption →
native code execution in workerd) therefore ends with the attacker holding the
**master internal token**, which grants:

- lasso-data with **arbitrary scoping headers** — every namespace's KV/R2/D1/queues,
  not just the worker's own;
- gate `/internal/bundles/:version_id` — **every worker's code and decrypted
  secrets**;
- `/__lasso/dispatch/*` on every runner.

Process-per-worker was adopted precisely to contain a compromised worker; this token
un-contains it. It converts "one worker compromised" into "platform compromised" —
strictly worse than the isolate-model status quo it replaced, because the loot is
now in-process rather than behind a supervisor channel.

**Fix (cheap, do it in v1):**

1. **Split tokens by audience**: gate-internal (supervisor↔gate), data-plane
   (worker→lasso-data), dispatch (gate/data→supervisor). A worker config never
   contains the gate-internal token — the supervisor is the only holder, which is
   what 06-security already claims.
2. **Scope the data-plane credential per worker**: gate mints a per-worker token
   (HMAC over worker id + granted resource ids with a data-plane key is enough — no
   DB lookup needed at lasso-data), rendered into that worker's config. lasso-data
   verifies token ↔ scoping-header consistency and rejects mismatches. Blast radius
   of a workerd escape becomes "that worker's own resources", which is the promise
   06-security.md makes.
3. Add to WP7.3's adversarial suite: *with* a worker's own token, attempt to read
   another namespace's KV / another worker's bundle. (The current suite only tests
   unauthenticated access.)

Cloudflare's equivalent: per-worker opaque keys held by the supervisor, handed to the
runtime per-request. Fly's equivalent (macaroon-style tokens scoped to one app/volume)
is the same idea. This is table stakes for the trust map in 01-architecture.md to be
honest.

### C2. Durable Object migrations are not designed at all

Wrangler's upload metadata carries a `migrations` array (`new_classes`,
`new_sqlite_classes`, `renamed_classes`, `deleted_classes`, `transferred_classes`,
with ordered migration tags). It appears in research/ecosystem.md's description of
the upload format — and then nowhere else: not in the bundle format (02-runner §4),
not in gate's schema (`do_namespaces(id, ns_id, class_key)`), not in 03-bindings.md,
not in the CLI doc, not in M5.

This is a correctness hole, not a missing feature:

- The platform generates `uniqueKey` per (namespace, class) and object IDs derive
  from it. A user renaming a DO class in their code (a routine refactor) uploads new
  metadata with a new class name → gate generates a **new uniqueKey** → every
  existing object ID resolves into an empty universe. **All DO data silently
  orphaned** on disk under the old key's directory. No error anywhere.
- Deleting a class similarly leaves unreachable SQLite files forever (no GC path is
  specified — see M10 below).
- Wrangler itself will refuse to deploy DO changes without a migration entry in some
  cases, so the wrangler-compat layer (M7) can't even pretend this doesn't exist.

**Fix:**

1. Gate schema: `do_migrations(worker_id, tag, seq)` — track applied migration tags
   per worker exactly like Cloudflare does; reject uploads whose migration list
   doesn't extend the applied sequence (out-of-order/replayed tags are the classic
   footgun).
2. Semantics per op: `new_classes`/`new_sqlite_classes` → mint uniqueKey (workerd is
   SQLite-backed for all DOs — accept both, document the equivalence);
   `renamed_classes` → **carry the existing uniqueKey to the new class name**;
   `deleted_classes` → tombstone the namespace, async-GC the `<uniqueKey>/`
   directory; `transferred_classes` (cross-script transfer) → reject with a clear
   error in v1.
3. Bundle format: carry `migrations` through to gate (the supervisor doesn't need
   it; it's control-plane state).
4. Roadmap: add to WP5.1/5.2 acceptance: "rename a DO class via migration → objects
   keep their data; upload with missing/out-of-order migration tag is rejected."

### C3. Alarms don't fire in a lazily-spawned world

Verified facts the plan already contains: workerd's per-namespace `AlarmScheduler`
lives **inside the workerd process**, and alarms "re-fire across process restarts"
(research/static-config-verification.md §5) — *when the process is running*.

But the runner model spawns every worker process **lazily on first request**
(01-architecture, 02-runner), and the do pool is exempt only from *reaping*, not
from lazy spawn. 08-kubernetes.md says after a do-pod restart "alarms catch up on
start" — of the pod, but nothing starts the per-worker processes. Sequence:

1. DO sets `storage.setAlarm(now + 1h)`; alarm persisted in `metadata.sqlite`. ✅
2. Node reboots / do pod reschedules / workerd image upgrade rolls the pod.
3. New supervisor starts with **zero child processes**; the DO worker gets no HTTP
   traffic (it's a cron-like scheduler DO — the whole point of alarms).
4. The alarm never fires. Silent, unbounded delay — and the design's own fallback
   plan for cron (research §4: "DO-alarm scheduler" escape hatch) would sit on this
   same broken foundation.

**Fix:** the do-pool supervisor must **eagerly spawn every DO-declaring worker's
process at pod start** (the set comes from the route feed's pool assignments; the
`/data/do/*` directories are a crash-safe local hint). This also warms the pool's
working set, which the no-reap default already implies is the intent. Add to WP5.1
acceptance: "alarm set → do pod deleted → pod rescheduled → alarm fires without any
inbound request." The current M5 test ("storage + alarms survive do-pod restart")
will pass with a request-driven test harness and still miss this bug — make the
no-traffic case explicit.

### C4. Two sources of truth for "which version serves": gateway header vs supervisor flip

The deploy flow gives both the gateway (`X-Lasso-Worker: ns/name@version` resolved
from its route table) and the supervisor ("atomically flip local routing
`ns/name → @newversion`") authority over version selection, without defining
precedence or the failure matrix:

- **Stale gateway**: a gateway partitioned from the route feed keeps sending
  `@old` after supersession. Does the supervisor honor the header and *cold-spawn a
  superseded version*? Nothing in 02-runner forbids spawn-on-miss for non-active
  versions — as written, a lagging gateway can pin old code live indefinitely, and
  the supervisor's "flip" is dead code (the header always names the version).
- **DO swap race (safety-critical)**: during drain-then-start, a request for `@new`
  arriving while `@old` still drains must NOT trigger spawn-on-miss — two live
  processes on one DO directory is the exact split-brain research §5 forbids. The
  503+Retry-After window is described, but spawn-on-miss + swap are not stated to be
  serialized per worker.

**Fix (spell out in 02-runner §1.1/1.2):**

1. The supervisor is authoritative: it spawns **only the version it believes
   active** (per its route feed state). Header version ≠ local active → `409`/`503
   X-Lasso-Error: version-skew`; gateway re-resolves and retries once. A gateway
   that can't converge is shed, not obeyed.
2. All per-worker lifecycle transitions (spawn-on-miss, swap, drain, reap) run under
   a **per-worker mutex**; DO-mode swap holds it across the whole
   unroute→SIGTERM→exit→spawn sequence so spawn-on-miss can never interleave.
3. Blue-green overlap window: old process keeps serving in-flight only; new requests
   for `@old` during overlap get routed to `@new` (CF gradual-deployment semantics
   are per-request probabilistic; ours is a hard cutover — say so).
4. Add to WP1.1/WP2.3 acceptance: stale-gateway test (feed-partitioned gateway must
   not resurrect a reaped superseded version) and a DO swap-race test under
   concurrent load asserting single-process invariant via supervisor logs.

---

## 2. The Durable Objects deep dive (the stated worry — and it's justified)

The v1 DO design is *correct* under its stated constraints (single replica, one
process per DO worker, drain-then-start). The problems are the constraints'
consequences, which the plan under-explores: durability is a bare PVC, availability
is a single pod with minutes-long failure modes, and scalability is "one pod holds
every DO worker forever". Below: what Cloudflare actually does, what Fly.io's
engineering contributes, and a concrete staged path.

### 2.1 What Cloudflare does internally (ground truth)

*(Filled in from commissioned research — see §2.1 sources.)*

### 2.2 What Fly.io's engineering offers (researched)

Fly ran this design's exact problem — single-writer SQLite pinned to one machine's
disk — for years, in production, and their trajectory is instructive
([Introducing LiteFS](https://fly.io/blog/introducing-litefs/),
[Litestream revamped](https://fly.io/blog/litestream-revamped/),
[Sprites design](https://fly.io/blog/design-and-implementation/)):

- **LiteFS** (FUSE filesystem, transaction-shipping via LTX files, Consul-lease
  leader election, `fly-replay`-based write forwarding proxy): the availability
  answer — hot replicas + failover in ~lease-TTL seconds. **But**: replication is
  async (unreplicated tail lost on hard primary failure), leadership is
  **per-cluster, not per-database** (one primary owns *all* DBs in the mount — a
  poor match for thousands of per-object SQLite files each ideally owned where the
  object runs), FUSE overhead, Consul dependency — and it's effectively
  **maintenance-mode** (LiteFS Cloud sunset Oct 2024; a handful of commits since;
  Fly's own words: "Litestream is the more popular project… easier to reason
  about"). **Recommendation: do not build on LiteFS.** The owner's instinct that
  "something like LiteFS might help" is right about the *problem class*, but the
  specific tool has been superseded by its own author.
- **Litestream v0.5+** (Oct 2025 rework; v0.5.16 current Aug 2026) is the better
  fit, almost point-for-point: LTX-based WAL shipping to S3-compatible storage,
  **multi-database support designed for directories with hundreds/thousands of
  DBs**, **dynamic register/unregister** of databases as files appear (exactly the
  one-SQLite-file-per-DO layout), tiered compaction + point-in-time restore,
  **follow mode** (`-f`) for a continuously-restoring warm standby, and
  **S3-conditional-write leases** for single-writer enforcement
  ([Litestream v0.5.0](https://fly.io/blog/litestream-v050-is-here/),
  [releases](https://github.com/benbjohnson/litestream/releases)). Data-loss window
  = sync interval (~1s).
- **Volume/deploy semantics**: Fly documents plainly that canary/bluegreen **cannot
  be used with attached volumes** and single-machine volume apps have a deploy
  downtime window ([seamless deployments](https://fly.io/docs/blueprints/seamless-deployments/),
  [volumes](https://fly.io/docs/volumes/overview/)). lasso's drain-then-start
  downtime for DO workers is the same honest accepted cost — good company. Their
  highest-leverage mitigation trick: **graceful lease/ownership handoff on SIGTERM**
  — the outgoing primary demotes/flushes/releases *before* dying so the successor
  promotes instantly instead of waiting for a lease TTL. That's the shape lasso's
  post-v1 multi-replica do pool should copy (and the `owner`/`generation` fields
  reserved in gate's schema anticipate it).
- **fly-replay** ([dynamic request routing](https://fly.io/docs/networking/dynamic-request-routing/)):
  proxy-level "re-run this request on machine X" response header, with an explicit
  **1 MB body-replay cap**. Direct prior art for two things in this plan: the
  gateway's retry-once behavior (which currently ignores request-body buffering —
  see M20) and any future owner-routing for a sharded do pool (wrong-pod accepts,
  answers "replay at pod N", gateway re-issues).
- **Corrosion** ([repo](https://github.com/superfly/corrosion),
  [blog](https://fly.io/blog/corrosion/)): SWIM-gossip + CR-SQLite state
  distribution with SQL-query subscriptions; Fly's edge proxies build routing tables
  from it. Validates lasso's route-feed shape (eventually-consistent routing state,
  authoritative check at the target — which is exactly what C4's fix formalizes).
  Their production lessons transfer: blast-radius scoping and treating replicated-
  schema migrations as dangerous.
- **Sprites** (2025–26): Fly's newest stateful-sandbox platform dropped hardware
  pinning entirely — object storage authoritative, local NVMe as cache, metadata in
  SQLite made durable via Litestream. The industry direction for "many small
  stateful things" is *storage-authoritative-elsewhere, disk-as-cache* — worth
  keeping in mind for lasso v2+, and philosophically identical to Cloudflare's own
  SQLite-DO storage-relay design.

### 2.3 Recommended staged DO plan

**v1 (amend the current plan, small effort):**

1. C2 (migrations), C3 (eager spawn), C4 (swap lock) fixes above.
2. **Litestream sidecar on the do pool** (and lasso-data), shipping
   `/data/do/**/*.sqlite` to any S3-compatible target (MinIO in-cluster or
   off-cluster), using v0.5 multi-db + dynamic discovery. This upgrades DO
   durability from "the PVC" to "PVC + continuous off-node backup with PITR" for
   one sidecar and zero application changes — and it's the same mechanism the gate
   already gets via its optional Litestream sidecar, so the chart pattern exists.
   Document the restore drill (WP5.2 already has one — point it at Litestream
   restore instead of raw PVC copy).
3. Document the **SQLite-backup caveat**: raw VolumeSnapshots of live WAL-mode
   SQLite are crash-consistent at best; Litestream (or `sqlite3 .backup`) is the
   supported story. 08-kubernetes.md currently implies VolumeSnapshots are fine.
4. Document the **traffic consequence** of DO pinning: a worker that declares any DO
   namespace has *all* its HTTP traffic pinned to the single do pool — no
   horizontal scaling for its stateless routes. Recommend the split pattern
   (stateless front worker → service binding → DO-declaring backend worker) in
   03-bindings.md; it's also Cloudflare's recommended shape and it keeps the do
   pool small.

**v1.5 (scalability without distributed systems):** static sharding. do pool becomes
a StatefulSet with N replicas, each with its own PVC; gate assigns each DO-declaring
worker to a replica (stable, stored, visible in the route feed); the gateway routes
each DO worker to its assigned replica only. Uniqueness still holds by construction
(a worker's namespaces live on exactly one replica). This removes the
every-DO-worker-on-one-pod memory ceiling and shrinks the blast radius of a pod loss
to one shard — with zero coordination protocol. Rebalancing = drain worker on
replica A, move directory (or restore from Litestream), update assignment,
generation-bump. This intermediate step is missing from the plan (it jumps from
1-replica to wdl-style leases) and is much cheaper than the lease design.

**v2 (availability):** the reserved leases + generation-fence design, now informed
by Fly: per-shard **warm standby via Litestream follow mode**, ownership as an
S3-conditional-write or gate-DB lease, **SIGTERM = release lease + final sync before
exit** so planned deploys/upgrades fail over in ~a second rather than a TTL, and
fencing tokens checked by lasso-data/gateway so a stale owner fails closed.

*(Cloudflare-internal comparison for §2.1 and the deploy-semantics divergence table
will be finalized from the commissioned research below.)*

---

## 3. High-priority findings

### H1. Service-binding target versions are frozen at config render time

02-runner §2 renders `X-Lasso-Worker: <target ns/name@pinned-active>` into the
caller's config. Worker processes live until *their own* next deploy or reap, so
when the *target* redeploys, every existing caller keeps addressing the superseded
version — and under spawn-on-miss will happily respawn it. Cloudflare service
bindings always hit the target's latest active version. As designed, `blog/api`
deploys a fix and `blog/frontend` keeps invoking the vulnerable version for hours.
It also defeats the reaper (superseded versions stay hot) and C4's version-skew
rejection (the stale header is *rendered in*, not transient).

**Fix:** render the target as `ns/name` (no version) and let the supervisor resolve
the active version at proxy time — it already holds the routing map; this also makes
service-binding calls follow deploys atomically with local flips. If per-invocation
pinning is ever wanted (CF gradual-deployment affinity), that's a runtime concern,
not a render-time constant.

### H2. Secret rotation breaks the process-identity invariant

Versions are immutable and secrets live outside the version (worker-scoped table);
`lasso secret put` "triggers a redeploy of the active version" (03-bindings.md). But
the redeploy produces the **same `ns/name@version`** with different rendered config
— and every mechanism in the runner (prewarm, swap, routing, socket naming, reap)
keys on version id. As specified, the supervisor has no reason to believe anything
changed and no way to address two generations of the same version.

**Fix:** introduce a **config generation** in process identity:
`ns/name@version+g<N>` where N increments on secret change (and any future
non-version config change, e.g. re-keyed C1 tokens). Route feed carries `(version,
cfggen)`; swap logic compares the tuple. Alternatively: synthesize a new version row
on secret rotation (simpler, but pollutes version history and rollback semantics —
the tuple is cleaner).

### H3. do-pool operational failure modes deserve first-class documentation

Single-replica StatefulSet on an RWO PVC means: node failure → PVC
detach/reattach → **minutes** of downtime for *every* DO-declaring worker (plus C3's
alarm silence, plus queued-alarm stampede on recovery); workerd image upgrade →
Recreate-style window for all DO workers simultaneously. 08-kubernetes.md mentions
the update window in one line; the node-failure case isn't discussed anywhere, and
the alarm-recovery interaction isn't either. Fly documents the equivalent honestly
("if the drive fails, that instance of your app goes down; there's no way around
that") and the plan's own standard is "honest fidelity" — apply it here: add a
"do pool failure modes" table to 02-runner or 08-kubernetes with measured/expected
durations, and let §2.3's staged plan be the roadmap answer.

---

## 4. Medium findings and under-explained sections

- **M1 — Idle definition for the reaper.** LRU reaping must treat open WebSockets
  (including hibernating-API connections) and in-flight dispatches as busy;
  "idle" is currently undefined. One sentence + one test.
- **M2 — Per-worker CPU containment is pod-global only.** Heap is per-process
  (v8Flags) but CPU is only the pod cgroup: one tight-loop worker starves every
  co-resident worker's cold starts and the supervisor itself. Cloudflare both
  meters CPU and evicts hogs to dedicated processes. Cheap v1 options: per-child
  `nice`/`SCHED_OTHER` weight, or per-child cgroups via delegation (needs
  `cgroupns` + writable cgroupfs — doable but fiddly in k8s; document either the
  mechanism or the accepted gap in 06-security's table, which currently only covers
  *metering*, not *containment*).
- **M3 — DO data lifecycle on worker delete.** Worker tombstone → what happens to
  `/data/do/<ns>_<name>/`? Needs a stated GC/retention policy (and interacts with
  C2's deleted_classes).
- **M4 — Dispatch → pool routing.** Cron/queue dispatches must target the pool the
  worker is pinned to (do pool for DO workers). Implied by "pool assignments" in
  the route feed; make it explicit in 04-control-plane.
- **M5 — Gateway wall-clock timeout vs streaming.** Default 30 s wall-clock kills
  SSE/long-poll/streaming responses; Cloudflare has no wall-clock limit while the
  client stays connected. WebSockets are exempted but streaming HTTP is not
  addressed; `limits.wallMs` exists in bundle metadata but its interaction with the
  gateway default is unspecified. Decide + document (suggest: response-idle timeout
  rather than absolute wall clock once headers are sent).
- **M6 — Retry-once needs a body policy.** Gateway retries on
  `bundle-unavailable`/swap-503 — with what request-body handling? Buffer-to-N-MB
  (fly-replay caps at 1 MB), or retry only idempotent/bodyless requests. Unspecified.
- **M7 — `request.cf` contents.** The gateway sets `cfBlobHeader = X-Lasso-Cf`, but
  what the blob contains is never specified; real-world workers read
  `cf.colo/country/asn/tlsVersion`. Define a minimal honest set + document the rest
  as absent.
- **M8 — R2 unknown-length uploads.** "Buffer-to-disk" — which disk (tmpfs? PVC?),
  what cap, what backpressure? One paragraph in 03-bindings.
- **M9 — In-flight caps.** Mentioned once in the request flow, never sized or
  scoped (per worker? per pod? per gateway?).
- **M10 — Doc contradiction on `localDisk` and `--experimental`.**
  research/workerd.md §2 says `--experimental` gates `localDisk`;
  research/static-config-verification.md §5 (later, source-line-cited) says it does
  not (only `ephemeralLocal`). Reconcile (the latter is presumably right). Related:
  since the harness needs `service_binding_extra_handlers`, **every** worker process
  runs with `--experimental` unlocked; the only guard on user code is gate-side
  compat-flag rejection. Worth stating as an accepted risk in 06-security.
- **M11 — Bundle format lists `py` modules** while Python workers are post-v1;
  gate's feature matrix must reject them until then (also: Pyodide fetches packages
  from the internet at runtime unless pre-cached — future sizing note already in
  backlog, fine).
- **M12 — Runner behavior with gate down at pod start.** Gateways persist a route
  snapshot to emptyDir; runners are stated to keep serving warm processes, but a
  *freshly restarted* runner pod with gate down has no feed and no snapshot — can it
  serve header-addressed spawns from its disk bundle cache? (Probably yes by
  design; C4's "supervisor spawns only what it believes active" makes the answer
  matter. Specify the degraded-start contract and add it to WP2.3's resilience
  test.)
- **M13 — Deploy safety is all-at-once.** CF runs health-mediated progressive
  rollouts internally; lasso flips 100% instantly on spawn-ready. Cheap v1 nicety:
  optional post-spawn smoke request (configurable path) before the flip; full
  gradual deployments stay post-v1 (already in backlog).

---

## 5. Roadmap amendments (concrete WP changes)

| WP | Add |
| --- | --- |
| WP0.2 spike | Assert alarm fires after process cold-start **with no inbound request** (C3); measure DO-swap unavailability window under load (503 duration). |
| WP1.1 | Per-worker lifecycle mutex + version-skew rejection tests (C4); reaper idle definition incl. WebSockets (M1). |
| WP2.1 | DO migrations: schema, tag sequencing, rename-preserves-uniqueKey, delete-GC, reject transfers (C2). Config-generation in identity tuple (H2). |
| WP2.2 | Route feed carries (version, cfggen) + pool assignment per worker (H2, M4). |
| WP2.3 | Stale-gateway test: partitioned gateway must not resurrect superseded versions (C4); retry body policy (M6); streaming-timeout policy (M5). |
| WP4.x | lasso-data validates per-worker token ↔ scoping header (C1). |
| WP5.1 | Eager spawn of DO workers at do-pod start + no-traffic alarm test (C3); Litestream sidecar for `/data/do` + restore drill (2.3); migration e2e (C2). |
| WP5.2 | DO dir GC on delete (M3); failure-modes doc (H3). |
| WP7.3 | Adversarial: cross-namespace access **with** a legitimate worker token (C1); dispatch forgery with data-plane token. |
| Post-v1 backlog | Insert "do pool static sharding (worker→replica assignment)" **before** "multi-replica do pool (leases + generation fences)" — it's the cheaper 80% (2.3). |

## 6. What the plan gets right (so it doesn't get re-litigated)

Process-per-worker over `workerLoader` (matches CF's own containment fallback and
removes the biggest experimental dependency); native binding shims with miniflare as
reference servers; immutable versions + pointer deployments; the gate-down-data-path-
survives contract; honest degradation for Cache/assets; conformance-gated workerd
pinning; drain-then-start for DO workers (the single-directory invariant is real and
verified); Gateway API with static routes. The research docs are genuinely strong —
the gaps above are almost all in the *seams between* documents, not in the documents
themselves.
