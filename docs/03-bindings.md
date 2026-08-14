# Bindings: KV, R2, D1, Durable Objects, queues, cron, secrets, cache, assets

> Revised 2026-08-14 for the process-per-worker model. User workers get **workerd's
> native binding shims**, declared in their process config and pointed at
> **lasso-data**, which implements the server side of each internal wire protocol.
> The exact protocols (headers, paths, response shapes, with source line references)
> are specified in research/static-config-verification.md §1 — that document is the
> implementation contract for this one. Miniflare's simulator workers are the
> maintained reference servers to diff against on every workerd upgrade.

Design rule: **API-shape compatibility with documented divergence.** The client side
is workerd's own C++ — byte-identical to production — so fidelity questions live
entirely in lasso-data's server behavior. Where a Cloudflare semantic isn't matched,
we do the honest thing and write it down; never silently approximate.

## lasso-data service (Go)

Single deployment (v1), one PVC, bearer-token auth, **speaking workerd's binding
protocols directly** (no translation layer in front). Binding scoping arrives as
platform-injected headers (`X-Lasso-KV-Namespace`, `X-Lasso-R2-Bucket`,
`X-Lasso-D1-Database`, `X-Lasso-Queue`) that user code can never see or forge
(config-injected; see 02-runner.md §2).

```
/data/
├── kv/<nsId>.sqlite            # one SQLite DB per KV namespace
├── d1/<dbId>.sqlite            # one SQLite DB per D1 database
├── r2/<bucketId>/
│   ├── meta.sqlite             # object metadata, multipart bookkeeping
│   └── blobs/ab/cd/<sha256>    # content-addressed bodies
├── cache/<worker>/…            # v1.5 cache backend (meta.sqlite + blobs)
└── queues/queues.sqlite        # all queues: messages, leases, retries, DLQ
```

SQLite via `modernc.org/sqlite` (pure Go), WAL mode, single writer per DB. Interfaces
allow Postgres (metadata) and S3/MinIO (R2 blobs) swap-in post-v1.

## KV (`kvNamespace` binding → KV protocol server)

- **Endpoints** (per research §1): `GET/PUT/DELETE /<key>?urlencoded=true` (+
  `expiration`/`expiration_ttl`/`cache_ttl` params, `CF-KV-Metadata` header),
  `POST /bulk/get`, `GET /?key_count_limit&prefix&cursor` for list — remembering
  list-response `metadata` must be a JSON-*string*; 404/410 = not-found; ignore
  `CF-KV-FLPROD-405`.
- **Storage**: `(key TEXT PK, value BLOB, metadata TEXT, expires_at INTEGER)`; lazy
  expiry + sweep. Limits at CF parity: values ≤ 25 MB, keys ≤ 512 B, metadata ≤ 1 KB.
- **Semantics**: strongly consistent (better than CF's eventual — documented
  divergence); `list` ordered by key with cursor pagination; `cacheTtl` accepted and
  ignored (no edge cache exists).
- **Not supported**: `deleteBulk` (JS-RPC, not HTTP — rejected with a clear error;
  documented).

## R2 (`r2Bucket` binding → R2 protocol server)

- **Protocol**: JSON-encoded `R2BindingRequest` (schema: workerd's `r2-api.capnp`,
  versioned, currently v1) in the `CF-R2-Request` header for reads / body prefix
  with `CF-R2-Metadata-Size` for writes; responses mirror the metadata-size +
  payload framing; errors via `CF-R2-Error` v4codes. Response shapes cross-checked
  against miniflare's `r2/schemas.worker.ts`.
- **v1 coverage**: head/get/put/delete/list (delimiters, prefixes, include
  options), ranged reads, onlyIf conditionals, full multipart lifecycle, checksums,
  http/custom metadata. Streaming both directions (metadata prefix parsed, payload
  piped; known Content-Length required by workerd on writes — buffer-to-disk for
  unknown-length uploads).
- **Post-v1**: presigned URLs, S3-compatibility endpoint (metadata layout designed
  for it), storage classes (accepted, recorded, ignored).
- **Divergence**: durability = the PVC.

## D1 (wrapped `cloudflare-internal:d1-api` binding → D1 HTTP server)

- **Endpoints**: `POST /query?resultsFormat=…` (single or array = transactional
  batch), `POST /execute`, `POST /dump` (deprecated but cheap). CF-shaped result
  envelopes (`success`, `results`, `meta.duration/rows_read/rows_written/
  last_row_id/changes`, `served_by: "lasso-primary"`); errors as
  `{success:false, error}` → `D1_ERROR` in user code.
- **Do not set** the `d1_binding_jsrpc` compat flag (JS-RPC mode); HTTP mode is the
  default and the contract.
- **Sessions**: echo/advance the `x-cf-d1-session-commit-token` header trivially
  (single primary — every read sees latest).
- **Migrations**: `lasso d1 migrations apply` mirrors wrangler's flow.
- **Divergences**: no read replicas; batch caps kept at CF's 500 statements.

## Durable Objects (native `durableObjectNamespaces` — do pool only)

The pivot's biggest simplification: **no facets, no dynamic-class plumbing.** A
DO-declaring worker's own process declares its namespaces exactly as a normal
workerd deployment would:

- Config per 02-runner.md §2: `durableObjectNamespaces = [(className, uniqueKey,
  enableSql = true)]`, `durableObjectStorage = (localDisk = "do-disk")` on the pod's
  PVC at `/data/do/<ns>_<name>/`. `uniqueKey` = platform-generated stable secret per
  (namespace, class), stored in gate, **never rotated** (object IDs derive from it;
  losing it orphans data — it lives in gate's DB and the backup story).
- Verified facts (research §5): one SQLite file per object under
  `<dir>/<uniqueKey>/<actor-id>.sqlite`; alarms persist in the namespace's
  `metadata.sqlite` and **re-fire across process restarts**; `localDisk` does not
  require `--experimental`; `enableSql` exposes `storage.sql` (all DOs are
  SQLite-backed regardless).
- **Placement & uniqueness**: DO-declaring workers are assigned to the **do pool**
  (StatefulSet, 1 replica v1) and pinned there by the gateway. Identity holds
  because exactly one process serves the worker: same-uniqueKey processes with
  different directories are verified to be fully independent universes, and two
  processes must never share a directory — hence the **drain-then-start deploy
  rule** for DO workers (02-runner.md §1.2) and the single-replica constraint.
  Multi-replica failover (wdl-style leases + generation fences) is the post-v1
  path; `owner`/`generation` fields are reserved in gate's schema now.
- **Cross-worker DO access** (worker A binds worker B's namespace): v1 restriction —
  a DO namespace is usable only by its defining worker; other workers reach it via a
  service binding to that worker (CF's recommended pattern anyway). Lifting this
  needs a cross-process DO stub protocol we deliberately defer.
- **Version skew on deploy**: drain-then-start means the old process (and its live
  objects, in-memory state, and WebSockets) ends before the new one starts —
  stricter than CF's lazy per-object migration; documented.
- **WebSocket hibernation**: API accepted; connections survive idle but not process
  swaps (documented divergence).

## Queues (`queue` producer binding → queue protocol server + delivery loop)

- **Producer side** (protocol per research §1): `POST /message` (raw body,
  `X-Msg-Fmt: text|bytes|json|v8`, optional delay) and `POST /batch` (JSON,
  base64-serialized bodies); responses carry the
  `{metadata:{metrics:{backlogCount…}}}` envelope; errors via
  `CF-Queues-Error-Code`. v8-format bodies are stored opaque and round-tripped
  intact (we never deserialize them).
- **Delivery**: lasso-data's per-queue loop batches per consumer settings
  (`max_batch_size`, `max_batch_timeout`, `max_retries`, `dead_letter_queue`),
  POSTs to a runner's dispatch endpoint → harness →
  `env.USER.queue(queueName, messages)` with `{id, timestamp, attempts,
  body|serializedBody}` per message (miniflare's broker is the reference for both
  shapes), applies the returned `FetcherQueueResult` (ack / retryAll / per-message
  retry, honoring `retryDelay`).
- Consumers are platform workers only (no HTTP push) in v1.

## Cron triggers

- Gate stores `crons` from wrangler config; its scheduler (robfig/cron, UTC — CF
  parity) fires dispatches → runner → harness →
  `env.USER.scheduled({cron, scheduledTime})`; outcome recorded (visible in tail +
  `lasso deployments`), no auto-retry (CF parity). Catch-up policy after gate
  downtime configurable (skip vs run-once).
- This is the platform's one use of the experimental
  `service_binding_extra_handlers` flag (harness-only). Fallback if upstream breaks
  it: native DO-alarm scheduler driving a synthetic HTTP route (documented, lossy —
  changes handler signature) — see research §4's escape hatch.

## Secrets

Unchanged flow, simpler delivery: AES-256-GCM at rest in gate (master key in a k8s
Secret; rotation re-encrypts); write-only API; decrypted only into the
supervisor-fetched bundle and rendered as plain `text` bindings in the per-process
0400 tmpfs config. Rotation takes effect on next process swap (`lasso secret put`
triggers a redeploy of the active version by default).

## Cache API (`cacheApiOutbound`)

- **v1**: omit `cacheApiOutbound` from rendered configs — workerd's default is a
  no-op passthrough (honest miss on every `match`).
- **v1.5**: point it at lasso-data's cache server implementing the verified
  protocol: `GET` with `Cache-Control: only-if-cached` → `CF-Cache-Status:
  HIT|MISS`; `PUT` with a full serialized HTTP response as body; `PURGE` method for
  delete; `CF-Cache-Namespace` header for named caches. LRU over SQLite+blobs,
  honoring `Cache-Control`. Never fake hits.

## Static assets

v1: assets ship in the bundle; the harness serves them from a `disk` service, and
the user worker gets an `ASSETS` service binding to the harness route (supports
`not_found_handling` basics). v2: dedicated content-addressed asset store in
lasso-data with separate upload (lifts bundle-size pressure, enables
`run_worker_first` semantics).

## Service bindings

`services = [{binding, service, entrypoint}]` → native `service` binding whose
target is an `external` service through the supervisor with a pinned
`X-Lasso-Worker` target header (02-runner.md §2). Fetch fully supported; JSRPC
between workers rides workerd's HTTP-based RPC transport across the supervisor hop —
conformance-tested, with any unsupported RPC shapes failing loudly at call time.
Entrypoint selection is carried in the target designator; cross-pool targets hop via
the gateway path with identical semantics.

## Explicitly out of scope for v1

`ai`, `vectorize`, `hyperdrive`, `analytics_engine`, `browser`, `email`,
`dispatch_namespace`, `workflows`, `mtls_certificate`, `pipelines`, KV `deleteBulk`,
D1 JS-RPC mode. Gate rejects uploads declaring them with a clear error listing
supported types; metadata schema reserves the type names for later.
