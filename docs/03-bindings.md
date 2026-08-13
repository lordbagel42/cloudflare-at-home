# Bindings: KV, R2, D1, Durable Objects, queues, cron, secrets, cache, assets

workerd ships the *client shape* of every binding and none of the storage. lasso
implements storage in **lasso-data** (Go) and exposes it to user code through the
loader's `ctx.exports` entrypoint classes (see 02-runner.md §3.2). This doc specifies
each binding's v1 semantics, storage, and known divergences from Cloudflare.

Design rule: **API-shape compatibility with documented divergence.** If we can't match
a Cloudflare semantic cheaply, we keep the method signature, do the honest thing, and
write the divergence down here and in user docs — never silently approximate.

## lasso-data service (Go)

Single deployment (v1), one PVC, internal REST API (bearer-token auth), storage
layout:

```
/data/
├── kv/<nsId>.sqlite            # one SQLite DB per KV namespace
├── d1/<dbId>.sqlite            # one SQLite DB per D1 database
├── r2/<bucketId>/
│   ├── meta.sqlite             # object metadata, multipart bookkeeping
│   └── blobs/ab/cd/<sha256>    # content-addressed bodies
└── queues/queues.sqlite        # all queues: messages, leases, DLQ
```

SQLite via `modernc.org/sqlite` (pure Go, no cgo), WAL mode, one write connection per
DB. Interfaces are defined so Postgres (control data) and S3/MinIO (R2 blobs) can be
swapped in post-v1 without touching the loader-facing API.

## KV

- **API**: `get(key, {type, cacheTtl})`, `put(key, value, {expiration, expirationTtl,
  metadata})`, `delete`, `list({prefix, limit, cursor})`, `getWithMetadata`. Values ≤
  25 MB (CF parity), keys ≤ 512 B, metadata ≤ 1 KB.
- **Storage**: SQLite table `(key TEXT PRIMARY KEY, value BLOB, metadata TEXT,
  expires_at INTEGER)`; lazy expiry on read + periodic sweep.
- **Semantics**: strongly consistent (better than CF's eventual consistency — allowed
  divergence, document it). `list` is ordered by key (CF parity, unlike wdl's
  unordered HSCAN). `cacheTtl` accepted and ignored (single-region: no edge cache to
  control).

## R2

- **API v1**: `head/get/put/delete/list`, ranged reads, conditional
  (`etag`) operations, `createMultipartUpload`/`uploadPart`/`complete/abort`,
  `checksums`, `httpMetadata`/`customMetadata`. Presigned URLs and the S3-compat
  endpoint are **post-v1** (the metadata layout is designed so an S3 gateway can be
  bolted on).
- **Storage**: metadata rows in `meta.sqlite`; bodies content-addressed on disk
  (dedup for free; GC by refcount). Streaming reads/writes end to end (no full
  buffering in the loader — R2 entrypoint pipes bodies).
- **Divergences**: single-region durability = the PVC's durability; no storage
  classes; `list` delimiter/prefix supported, `include` options supported.

## D1

- **API**: `prepare().bind().first/all/raw/run()`, `batch()` (transactional),
  `exec()`, `withSession()` accepted with session semantics = no-op (single primary).
- **Implementation**: the loader's D1 entrypoint sends `{sql, params}[]` to
  lasso-data, which executes against the database's SQLite file and returns
  CF-shaped result envelopes (`results`, `meta.duration`, `served_by`,
  `rows_read/written`, `last_row_id`, `changes`). Reference for the wire shapes:
  miniflare's D1 simulator and CF docs.
- **Migrations**: `lasso d1 migrations apply` mirrors wrangler's flow (a
  `d1_migrations` table + SQL files).
- **Divergences**: no read replicas; 500-statement batch cap kept; auth model is
  per-binding not per-token.

## Durable Objects

The hardest binding; the design follows wdl's proven shape (facets + localDisk +
single-owner routing) minus the multi-replica leases we don't need at v1.

- **Placement**: all DO namespaces live in the **do pool** (StatefulSet, 1 replica
  v1). Its loader variant declares one config-level supervisor DO namespace
  (`uniqueKey` = platform-generated stable secret, `enableSql = true`,
  `durableObjectStorage = localDisk` on the PVC).
- **Instantiation**: user worker calls `env.MY_DO.get(id)` → loader DONamespace stub
  (props: doNsId, className) → HTTP to do pool with `(doNsId, className, objectId,
  versionId)` headers → do-pool loader routes to the supervisor DO for that
  `(doNsId, objectId)` → the supervisor DO loads the *user worker's version bundle*
  via its own `workerLoader`, extracts `stub.getDurableObjectClass(className)`, and
  runs it as `ctx.facets.get(objectId-scoped-name, …)`. The facet gets its own SQLite
  storage, invisible to the supervisor DO's own data (upstream-documented facet
  guarantee).
- **Identity**: `idFromName`/`newUniqueId`/`idFromString` implemented in the loader
  stub (deterministic hashing with the namespace key, CF id format). Uniqueness holds
  because exactly one pod owns a namespace (StatefulSet + gateway routing); v2 can add
  wdl-style leases + generation fences for multi-replica failover — schema fields for
  `owner`/`generation` are reserved from day one.
- **Version skew rule** (CF parity): an in-flight object keeps its loaded version
  until it idles out; new activations use the active version. Facet abort on version
  activation is a tunable (default: lazy).
- **Alarms**: native (facet storage supports `setAlarm`); verify-on-upgrade in
  conformance (alarm persistence across process restarts on localDisk).
- **WebSockets**: pass through (gateway → runner → do pool). WebSocket *hibernation*
  is accepted-API/degraded-semantics: connections survive idle but not process
  restarts; documented divergence.
- **Storage risk flag**: `localDisk` DO storage is marked experimental upstream;
  mitigations: PVC snapshots (document a backup cron), conformance tests on upgrade,
  and the storage directory format (one SQLite file per object) is simple enough to
  migrate mechanically if upstream changes it.

## Queues

- **Producer API**: `send()`, `sendBatch()`, content types json/text/bytes/v8. (v8
  serialization round-trips through the loader as bytes — divergence: structured
  clone fidelity is conformance-tested, falls back to json with a warning.)
- **Storage/delivery**: lasso-data owns `queues.sqlite` (messages, visibility
  leases, retries, DLQ). A delivery loop in lasso-data leases batches per consumer
  worker, POSTs to a runner dispatch endpoint (`/__lasso/dispatch/queue`), applies
  the returned `FetcherQueueResult` (ack / retryAll / per-message retry), honoring
  `max_batch_size`, `max_batch_timeout`, `max_retries`, `dead_letter_queue` from
  wrangler config.
- **Consumers** are workers in the same platform (no HTTP push consumers in v1).

## Cron triggers

- Gate stores `crons` per worker version (from wrangler config `triggers.crons`);
  scheduler (robfig/cron, UTC) enqueues dispatches with at-least-once semantics and
  a jittered catch-up policy after gate downtime (configurable: skip vs run-once).
- Dispatch → runner `/__lasso/dispatch/scheduled` → `stub.scheduled({cron,
  scheduledTime})`; failures logged + visible in `lasso deployments`/tail; no
  automatic retry (CF parity).

## Secrets

- `lasso secret put NAME` (or wrangler-compat `PUT .../secrets`) → value encrypted
  (AES-256-GCM, master key from a k8s Secret generated at install) → stored in gate
  DB; write-only thereafter (CF parity: list shows names only).
- Delivery: gate decrypts when building a bundle envelope; envelopes travel over
  in-cluster TLS/authed channels and are cached on runner disk **encrypted at rest**
  with a per-pool key delivered via downward secret mount (v1 simplification:
  cache-in-tmpfs-only for envelopes with secrets; disk cache holds modules only).
- Rotation: new secret value → new envelope build; takes effect on next version
  activation or isolate reload (documented; CF has the same "redeploy to apply"
  shape).

## Cache API (`caches.default`)

v1: honest no-op (every `match` misses, `put` accepted and dropped) — matching
workerd's default behavior without a `cacheApiOutbound`. v1.5: wire
`cacheApiOutbound` to a small LRU cache service in lasso-data (SQLite+blob, honoring
`Cache-Control`). Never lie about hits.

## Static assets (Workers Assets)

v1: bundle-embedded assets served by the loader's Assets stub (fine for small sites;
32 MB bundle cap applies). v2: dedicated asset store in lasso-data (content-addressed,
uploaded separately like CF's assets flow) to lift the cap and enable
`run_worker_first`, `not_found_handling` config.

## Service bindings

`services = [{binding, service, entrypoint}]` in wrangler config → loader
ServiceBinding stub with `props {target, entrypoint}`. Calls re-enter the loader
in-process: resolve target's active version (cached route info, refreshed via gate
SSE), `getWorker(target).getEntrypoint(entrypoint)`, forward fetch/RPC. JSRPC between
user workers is supported to the extent Fetcher RPC supports it (conformance-tested);
cross-pool targets hop via HTTP with the same header contract as the gateway.

## Explicitly out of scope for v1

`ai`, `vectorize`, `hyperdrive`, `analytics_engine`, `browser`, `email`,
`dispatch_namespace`, `workflows`, `mtls_certificate`, `pipelines`. Gate rejects
uploads that declare them, with a clear error listing supported binding types. (The
metadata schema keeps their type names reserved so support can be added without
breaking stored versions.)
