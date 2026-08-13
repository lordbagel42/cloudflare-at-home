# lasso-gate: the control plane

Single Go service owning identity, storage of record, deploy pipeline, scheduling,
and the public API. Deliberately boring: SQLite + blob files on one PVC, one replica,
stateless-restartable, with the property that **the data path survives gate downtime**
(gateways and runners cache what they need; only deploys, cron, tail, and admin pause).

## Data model (SQLite, `gate/store`)

```sql
namespaces(id, name UNIQUE, isolation TEXT DEFAULT 'shared', created_at)
workers(id, ns_id, name, created_at, UNIQUE(ns_id, name))
versions(id ULID, worker_id, content_hash, metadata JSON, created_at,
         created_by, source TEXT)          -- immutable; never UPDATE
modules(version_id, name, type, sha256, size)          -- content in blob store
deployments(worker_id PRIMARY KEY, version_id, updated_at, updated_by)
routes(id, pattern, worker_id, kind='hostname'|'default', created_at)
secrets(worker_id, name, ciphertext, nonce, updated_at, PRIMARY KEY(worker_id,name))
kv_namespaces(id, ns_id, title)            r2_buckets(id, ns_id, name)
d1_databases(id, ns_id, name)              do_namespaces(id, ns_id, class_key)
queues(id, ns_id, name, settings JSON)
crons(worker_id, expr, PRIMARY KEY(worker_id, expr))
tokens(id, name, hash, scopes JSON, ns_id NULL, created_at, last_used)
audit(id, at, token_id, action, subject, detail JSON)
```

Blob store: `blobs/ab/cd/<sha256>` module contents, refcounted GC. DB + blobs on the
gate PVC; optional Litestream sidecar for continuous backup (Helm value).

Invariants:

- `versions` rows and blobs are immutable and content-addressed (dedup across
  versions is automatic — unchanged modules share blobs).
- `deployments` is the only mutable pointer; every change is audited and produces a
  route-feed event. Rollback = write an old version id (validated to exist).
- Deleting a worker tombstones it (route removal is immediate; blob GC is async).

## APIs

Three surfaces on one listener (`api.<base-domain>`, TLS at the cluster Gateway):

### 1. Platform API (`/v1/…`) — the native, canonical surface

```
POST   /v1/namespaces                          create ns
GET    /v1/namespaces/:ns/workers              list
PUT    /v1/namespaces/:ns/workers/:name/versions    multipart upload → version (option ?activate=true)
POST   /v1/namespaces/:ns/workers/:name/deployments {version_id}   activate/rollback
GET    /v1/namespaces/:ns/workers/:name/versions    history
DELETE /v1/namespaces/:ns/workers/:name
PUT    /v1/…/:name/secrets/:secret             write-only secret
GET/PUT/DELETE /v1/…/kv/:nsId/…                data-plane admin (list/get/put keys) → proxied to lasso-data
POST   /v1/…/:name/tail                        start tail session → SSE/WebSocket stream
GET    /v1/routes … PUT /v1/routes             custom hostnames
GET    /v1/healthz, /v1/metrics
```

Upload body = the same multipart format as Cloudflare (metadata part + module parts)
so the CLI and the compat layer share one ingest path. Validation at ingest: supported
binding types only, referenced resources exist (or `--create-missing` for
kv/r2/d1/queues, mirroring wrangler's provisioning UX), compat date ≤ pinned workerd,
size caps, reserved names (`__Lasso*`), env-size preflight.

### 2. Wrangler-compat API (`/client/v4/…`) — a translation layer

Enough for stock `wrangler deploy|versions|rollback|secret|kv|tail` pointed at
`CLOUDFLARE_API_BASE_URL=https://api.lasso.lan/client/v4` with a lasso token:

- `GET /user/tokens/verify`, `GET /accounts`, `GET /memberships` — static shims
  (account id ↔ namespace).
- `PUT /accounts/:a/workers/scripts/:name` (multipart) → version+activate.
- `GET/POST …/scripts/:name/…` settings, subdomain, secrets, tail (the tail endpoint
  speaks wrangler's expected protocol), KV namespace CRUD + bulk ops.
- Anything unimplemented returns a structured CF-style error with a docs pointer.

This layer is a **milestone after** the native API works; it only translates, never
stores.

### 3. Internal API (`/internal/…`, cluster-only, bearer token)

```
GET  /internal/bundles/:version_id          envelope (built on demand, cached; secrets decrypted here)
GET  /internal/routefeed                    SSE: full snapshot + deltas {routes, deployments, pools}
POST /internal/tail                         tail event ingest from loaders
POST /internal/dispatch-callback            cron/queue outcome reporting
```

## Deploy pipeline (inside gate)

1. Parse multipart → normalize metadata (one internal schema regardless of surface).
2. Validate (see above) + compute content hash over sorted module hashes + metadata.
3. Idempotency: same content hash + same worker → return existing version (no-op
   deploys are free).
4. Store blobs, insert version, (optionally) flip deployment, emit route event,
   audit.
5. Envelope build is lazy (first `/internal/bundles/` hit) and cached in memory +
   disk; envelopes with secrets are marked non-disk-cacheable for runners (see
   06-security.md).

## Scheduler (cron) & queue delivery coordination

- Cron: single-replica gate runs robfig/cron over the `crons` table (UTC; CF
  supports UTC only — parity). Each tick POSTs a dispatch to a runner via the pool
  service with retry-once-on-another-pod; outcomes recorded to `audit` + tail.
- Queue delivery loops live in lasso-data (closer to the data); gate only provides
  consumer→worker resolution via the route feed. (Decision D11: keeps gate free of
  hot loops; data already owns leases/retries.)

## Route feed & caching contract

Gateways and runners must keep serving through gate restarts:

- On connect: full snapshot (all hostname routes, worker→active-version map, pool
  assignments) with a monotonic sequence number; then deltas. Reconnect with
  `Last-Event-ID` → resync from snapshot if the gap is too large.
- Gateways persist the last snapshot to an emptyDir so a *simultaneous*
  gateway+gate restart still comes up serving (stale-but-working routes) before gate
  returns.

## AuthN/Z

- Tokens: `lasso token create --scope deploy --ns blog` → random 32-byte secret,
  stored hashed (argon2id). Scopes: `admin`, `deploy`, `read`, `data` × optional
  namespace restriction. The CLI stores its token in the OS keychain or
  `~/.config/lasso/`.
- Bootstrap: Helm generates an admin token into a k8s Secret; `lasso login` walks
  the user through it.
- All internal services use a chart-generated internal bearer token (rotated by
  reinstall/upgrade; mTLS is a post-v1 hardening option).

## Web UI (post-v1)

A small dashboard (workers list, versions, deploy/rollback button, live tail, KV/R2
browsers) served by gate as static files, talking to `/v1`. Explicitly after the CLI
is solid.
