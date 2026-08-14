# Research: static-config verification (process-per-worker facts)

> Research date: 2026-08-14, commissioned for the pivot away from `workerLoader`.
> Sources: `cloudflare/workerd` @ `3d43d41` and `cloudflare/workers-sdk` @ `fb6b51b`
> (paths relative to those repos), plus direct measurements with the
> `workerd@2026-08-14` npm binary on a 4-vCPU x86-64 Linux VM. **This is the
> implementation reference for lasso-data's binding protocol servers and the
> supervisor's process lifecycle.**

## 1. Native binding shim protocols (client = workerd C++; server = lasso-data)

Every binding below is client-side "HTTP to a named service" in workerd, and
miniflare's `packages/miniflare/src/workers/{kv,r2,d1,queues,cache}` workers are the
exact server-side counterparts (they share header/param constants with workerd's
client code). The host in URLs is always a placeholder (`https://fake-host/`,
`http://d1/`) — route on path/query/headers only.

### kvNamespace — `src/workerd/api/kv.c++`

- **get / getWithMetadata** (kv.c++:365-494): `GET
  https://fake-host/<percent-encoded key>?urlencoded=true[&cache_ttl=N]`. Response:
  200 with raw value as body; **404 or 410 → key not found** (returns null);
  metadata in response header `CF-KV-Metadata` (JSON string); `CF-Cache-Status`
  optionally echoed to JS. Content-Encoding honored (auto-decoded).
- **bulk get** (kv.c++:180-303): `POST https://fake-host/bulk/get`, JSON body
  `{"keys":[...], "type"?: "text"|"json", "withMetadata"?: true, "cacheTtl"?: "N"}`.
  Response: JSON object mapping key → value (or `{value, metadata}` when
  withMetadata). Miniflare side: `kv/namespace.worker.ts:124-172`.
- **put** (kv.c++:578-710): `PUT
  https://fake-host/<key>?urlencoded=true[&expiration=unixSecs][&expiration_ttl=secs]`,
  body = raw value bytes, request header `CF-KV-Metadata: <JSON>` when metadata
  given. Any 2xx = success.
- **delete** (kv.c++:712-744): `DELETE
  https://fake-host/<encodeURIComponent(key)>?urlencoded=true`.
- **list** (kv.c++:496-576): `GET
  https://fake-host/?key_count_limit=N&prefix=...&cursor=...`. Response JSON:
  `{keys:[{name, expiration?: unixSecs, metadata?: "<JSON-string>"}],
  list_complete: bool, cursor?: string}` — **`metadata` must be a JSON-*serialized
  string*, which workerd re-parses** (kv.c++:79-81; miniflare comments the same).
- Extra: workerd sends `CF-KV-FLPROD-405: <url>` on every request — ignore
  server-side. Errors: any non-2xx (other than 404/410 on GET) throws
  `KV <METHOD> failed: <status> <statusText>`.
- **Caveat:** `deleteBulk` (kv.c++:746) is **JS-RPC**, not HTTP — a plain-HTTP shim
  won't support it (optional; skip in v1 and document).

### r2Bucket — `src/workerd/api/r2-rpc.c++`, `r2-bucket.c++`, `r2-api.capnp`

- Every op serializes an `R2BindingRequest` capnp struct **to JSON** (capnp JSON
  codec), discriminated by `"method"`: one of
  `head|get|put|list|delete|createBucket|listBucket|deleteBucket|createMultipartUpload|uploadPart|completeMultipartUpload|abortMultipartUpload`
  with flattened per-op fields (`r2-api.capnp:15-31`): `object`, `range
  {offset,length,suffix}`, `onlyIf` conditionals, `customFields`, `httpFields`,
  checksums, `storageClass`, `ssec`, etc.
- **Read ops (get/head/list)** — `doR2HTTPGetRequest` (r2-rpc.c++:78-155): `GET
  https://fake-host/` with the JSON in request header **`CF-R2-Request`**.
  Response: header **`CF-R2-Metadata-Size: N`**, body = N bytes of JSON result
  metadata followed immediately by the object payload bytes. Metadata capped at
  1 MiB.
- **Write ops (put/delete/multipart)** — `doR2HTTPPutRequest` (r2-rpc.c++:162-236):
  `PUT https://fake-host/` with request header `CF-R2-Metadata-Size: <len(JSON)>`;
  body = JSON metadata immediately followed by value bytes; `Content-Length` must be
  known (workerd requires a known stream length). Response body = JSON result.
- Errors: status ≥ 400 with header **`CF-R2-Error`**:
  `{"version":0,"v4code":<int>,"message":"..."}`.
- Miniflare server matches exactly: `r2/bucket.worker.ts:1014-1092` (decode paths at
  147-195); response JSON shapes are zod-schema'd in `r2/schemas.worker.ts`;
  constants in `r2/constants.ts`. R2's request schema is versioned
  (`r2-api.capnp` `version` field, currently 1).

### D1 — `src/cloudflare/internal/d1-api.ts` (wrapped binding over an inner fetcher)

Two modes, switched by compat flag `d1_binding_jsrpc` (do **not** set it). Default
HTTP mode:

- `POST http://d1/query?resultsFormat=ROWS_AND_COLUMNS|ARRAY_OF_OBJECTS|NONE` and
  `POST http://d1/execute?resultsFormat=NONE`. Body: single
  `{"sql": "...", "params": [...]}` or array (batch → `/query` with array).
- `prepare().all()/run()/raw()/first()` → `/query`; `exec()` → `/execute` with the
  query split by lines; deprecated `dump()` → `POST http://d1/dump` (raw bytes).
- Response: JSON, one result object per statement (array in ↔ array out):
  `{success: true, results: [...], meta: {duration, ...}}` or
  `{success: false, error: "..."}`; errors surface as `D1_ERROR: <message>`.
- Sessions: request header `x-cf-d1-session-commit-token` (bookmark or
  `first-primary`/`first-unconstrained`); server may return an updated token in the
  same response header. A single-primary shim can echo/ignore.
- Miniflare server: `d1/database.worker.ts:209-210`.

### queue producer — `src/workerd/api/queue.c++`

- **send()** (queue.c++:240-306): `POST https://fake-host/message`, body = raw
  serialized message bytes, `Content-Type: application/octet-stream`, headers
  `X-Msg-Fmt: text|bytes|json|v8` (default json under `queues_json_messages` flag,
  else v8) and optional `X-Msg-Delay-Secs: N`.
- **sendBatch()** (queue.c++:335-449): `POST https://fake-host/batch`, JSON body
  `{"messages":[{"body":"<base64 of serialized body>", "contentType"?: "...",
  "delaySecs"?: n}, ...]}`, informational headers `CF-Queue-Batch-Count`,
  `CF-Queue-Batch-Bytes`, `CF-Queue-Largest-Msg`.
- **metrics()**: `GET https://fake-host/metrics`.
- Response: **200** with JSON
  `{"metadata":{"metrics":{"backlogCount":n,"backlogBytes":n,"oldestMessageTimestamp":msEpoch|0}}}`
  for send/batch; metrics returns the bare metrics object. Errors: non-200 +
  headers `CF-Queues-Error-Code` / `CF-Queues-Error-Cause`.
- Miniflare server: `queues/broker.worker.ts:488-546` — matches byte-for-byte.

### cacheApiOutbound — `src/workerd/api/cache.c++` (miniflare `cache/cache.worker.ts`)

- **match()**: `GET <full original request URL>` with the request's headers plus
  `Cache-Control: only-if-cached`. Server **must** reply with `CF-Cache-Status`:
  `HIT` (body/status/headers = the cached response) or `MISS`/`EXPIRED`/`UPDATING`
  (= miss). Missing/invalid header = miss.
- **put()**: `PUT <request URL>`; the request **body is a full serialized HTTP
  response** (status line + headers + body) — server parses and stores.
- **delete()**: custom method **`PURGE <request URL>`** with `X-Real-IP: 127.0.0.1`.
  200 = deleted, 404 = not found, 429 = rate-limited (throws).
- **Named caches** (`caches.open(name)`): workerd adds header
  `CF-Cache-Namespace: <encodeURIComponent(name)>`; absent = default cache.

## 2. Isolate loading: eager at process startup

**Every config-declared worker is compiled and instantiated eagerly before any socket
listens** (`Server::run` awaits `startServices` before `listenOnSockets`,
server.c++:6788-6826; `makeWorkerImpl` synchronously creates the isolate, compiles,
runs top-level module evaluation, validates handlers, server.c++:5605+). No lazy path
exists for config-declared workers. Measured: 1 trivial worker → ~32 ms to first
response; 50 workers in one config → ~168 ms, +58 MB RSS (~2.8 ms / ~1.2 MB marginal
per trivial isolate). **Ideal for process-per-worker: each process has one user
worker plus tiny shim workers.**

## 3. Baseline footprint & startup (measured; no published numbers exist)

workerd 2026-08-14 npm Linux x64, 4-vCPU VM:

| Metric | Value |
|---|---|
| `workerd serve` start → first HTTP response (1 trivial worker) | **31–36 ms** |
| RSS, idle, 1 trivial worker | **~41–42 MB** |
| RSS after 200 requests | ~43–44 MB |
| RSS, 50 trivial workers, one process | ~100 MB |
| Startup, 50 trivial workers | ~165–171 ms |

Budget ~40–45 MB fixed per process + the app's own V8 heap; ~20–25 idle trivial
workers per GB. Expect ±30% by hardware; Python workers far heavier. Re-measure on
target hardware (roadmap M0 spike).

## 4. Cron/scheduled + queue delivery into config-declared workers

**Confirmed: workerd has no cron scheduler and no config-level scheduled/queue
trigger** (no `scheduled`/`cron` in workerd.capnp; `workerd test` only invokes
`test()` handlers). The only mechanism: a co-declared trusted worker with a `service`
binding to the target and compat flag `service_binding_extra_handlers`, calling
`fetcher.scheduled({scheduledTime, cron})` / `fetcher.queue(queueName, messages,
metadata)` (`src/workerd/api/http.h:418-433`, gated methods). The flag is
`$experimental` (compatibility-date.capnp:295-302, with the V8-deserializer warning
on `queue()` `serializedBody`) → **`workerd serve` must run with `--experimental`**.
Miniflare always passes `--experimental`.

How miniflare does cron: its entry worker (flags include
`service_binding_extra_handlers`) routes magic URL **`/cdn-cgi/local/scheduled`** to
`service.scheduled(...)` on the target's binding (`workers/core/entry.worker.ts:554`,
`workers/core/scheduled.ts`); wrangler runs **no timer** — it tells the user to curl
that URL. Queue delivery in miniflare: broker DO batches producer POSTs then calls
`service.queue(name, messages, metadata)` with `{id, timestamp, attempts,
body|serializedBody}` and applies retries/DLQ from the returned `FetcherQueueResult`
(`workers/queues/broker.worker.ts:257-385`) — the pattern for lasso's delivery.

**Non-experimental escape hatch:** DO **alarms fire natively with no flag** — a tiny
co-declared DO re-arming `storage.setAlarm()` can drive cron-like scheduling via
ordinary `fetch()`; the experimental flag is only needed to invoke the user's real
`scheduled()` handler.

## 5. Config-declared DOs with localDisk: confirmed

- Config: `durableObjectNamespaces = [(className, uniqueKey, enableSql = true)]` +
  `durableObjectStorage = (localDisk = "<disk service>")` (writable DiskDirectory).
  localDisk is doc-marked experimental **but does not require `--experimental`**
  (only `ephemeralLocal` does — server.c++:6928-6954).
- Layout: `<diskRoot>/<uniqueKey>/` per namespace; **one SQLite file per object**
  `<64-hex-actor-id>.sqlite` (+`-wal`/`-shm`); alarm state in `metadata.sqlite` per
  namespace directory.
- **Alarms persist and re-fire across restarts** (per-namespace `AlarmScheduler`,
  server.c++:404-418, 1713-1731).
- **Same uniqueKey in two processes → two fully independent universes** (IDs
  identical, files/state separate; zero coordination). And **never point two live
  processes at the same directory** — each holds its own ActorCache/output-gate and
  its own alarm scheduler over the same `metadata.sqlite`; nothing arbitrates.
  Consequence: DO-bearing workers must be **drain-then-start** on deploy (no
  blue-green overlap), pinned to one process.

## 6. Graceful swap: `--socket-fd`, SIGTERM drain, no SO_REUSEPORT

- `--socket-fd <name>=<fd>` fully supported: the supervisor must `bind()`+`listen()`
  first (workerd validates `SO_ACCEPTCONN`); workerd takes ownership.
- **Readiness**: with `--control-fd=N`, workerd writes
  `{"event":"listen","socket":<name>,"port":<p>}\n` per socket when actually ready
  (server.c++:7131-7139) — the readiness gate (miniflare relies on it too).
- **SIGTERM drains**: idle connections closed immediately, in-flight requests
  (including WebSockets) run to completion, accept loops cancel, process exits when
  empty. **No drain timeout exists** — the supervisor must SIGKILL after a grace
  period.
- **SO_REUSEPORT: workerd never sets it.** Sharing one supervisor-owned listening FD
  between old+new child during overlap works (kernel balances `accept()`); or a unix
  socket per child generation with supervisor-side switching (lasso's model — the
  supervisor is already the pod's router/proxy).

## Flags / unverified

1. §3 numbers are single-machine measurements, not a benchmark suite.
2. The binding wire formats are internal contracts, **not compat-date-stable** —
   pin workerd, re-diff `miniflare/src/workers/*` on every upgrade (miniflare is the
   maintained reference server).
3. KV `deleteBulk` and D1 `d1_binding_jsrpc` mode are JS-RPC, unsupported by
   plain-HTTP shims (both optional/non-default).
4. FD-passing overlap and SIGTERM drain verified from source, not end-to-end tested
   (standard kernel semantics; covered by the M0 spike).
