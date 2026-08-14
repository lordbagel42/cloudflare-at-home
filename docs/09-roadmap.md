# Roadmap: milestones and work packages

> Revised 2026-08-14 for the process-per-worker model.

Written to be executed by AI coding agents (Opus/Sonnet tier) or humans, one work
package (WP) at a time. Rules for executors:

- **Read first**: the WP's listed docs + research docs (especially
  research/static-config-verification.md — it contains the wire protocols and
  measured numbers this plan builds on). They contain settled decisions; don't
  re-litigate them in code. If reality contradicts the plan, fix the docs in the
  same PR and say so.
- **Every WP ends green**: code + tests + a runnable demo command in the WP's
  acceptance section. CI must pass. No WP leaves `make e2e` regressed.
- **Pin everything** (workerd, wrangler, toolchains) in `versions.lock.json`,
  created in M0.
- Conventions: Go 1.26+, `golangci-lint`; TypeScript strict, Node 22, pnpm;
  conventional commits; every service gets a `Dockerfile` + `make run-local`.

Milestones are sequential; WPs inside a milestone are parallelizable unless noted.

---

## M0 — Repo foundation + de-risking spike

**WP0.1 Scaffolding.** Monorepo layout per 00-overview.md; Makefile; Go workspace;
pnpm workspace; `versions.lock.json` (workerd pinned ≥ 1.20260619.1, newest passing);
Apache-2.0 LICENSE; CI (lint + unit + stub e2e); kind dev cluster script
(`hack/kind-up.sh`) with Envoy Gateway.
*Accept:* CI green; `hack/kind-up.sh` yields a Gateway-API-ready cluster.

**WP0.2 Process-model spike (seed of the conformance suite).** A `spike/` Go harness
that, against the pinned workerd binary:

1. Renders a per-worker text capnp config (user worker + harness worker) and spawns
   `workerd serve` on a unix socket; parses `--control-fd` readiness events.
2. Proves SIGTERM drain (in-flight request completes; hung request requires
   SIGKILL after grace — assert both).
3. Blue-green swap: spawn v2, flip a toy router, drain v1 — zero failed requests
   under concurrent load.
4. Native binding smoke: a stub HTTP server speaking the KV protocol
   (research §1) receives get/put/list from a `kvNamespace` binding; assert header
   and shape details (incl. list-metadata-as-JSON-string).
5. Dispatch: harness calls `fetcher.scheduled()`/`.queue()` on the user worker
   under `service_binding_extra_handlers`; assert outcome envelopes.
6. DO smoke: config-declared namespace with `localDisk`; object storage + alarm
   persists across process restart; two processes on different dirs are independent.
7. Measure and record (into `spike/RESULTS.md`): process RSS, startup-to-ready,
   swap latency on target hardware.

*Accept:* `go test ./spike/...` green on the pinned workerd. Findings that
contradict docs → update docs in the same PR.

---

## M1 — Runner (docs: 02-runner.md)

**WP1.1 Supervisor: process manager + router.** Config template rendering (all
binding kinds, scoping-header injection, secrets as text bindings, 0400 tmpfs),
spawn/ready/route/drain/reap state machine, blue-green vs drain-then-start (DO)
swap modes, LRU idle reaping, crash backoff + failing state, RSS watchdog,
unix-socket reverse proxy with WebSocket passthrough and internal-header hygiene,
bundle disk/tmpfs cache with hash verification, health + Prometheus endpoints
(per-worker-process RSS/CPU).
*Accept:* integration tests against real workerd: swap under load with zero failed
requests; kill -9 child → respawn; crash-loop isolation (other workers unaffected);
DO-mode swap never overlaps processes; watchdog reaps under synthetic pressure;
offline serving from warm cache.

**WP1.2 Harness worker.** TS, zero runtime deps: dispatch relay
(scheduled/queue → outcome envelopes), tail forwarding with sampling caps, asset
serving from the disk service. Built into the runner image.
*Accept:* wd-test/integration: dispatch outcomes round-trip (incl. retryAll and
per-message retries); tail batches arrive at a stub sink with correct worker
identity; assets serve with correct content types.

**WP1.3 Runner image + stubgate.** Distroless multi-arch image (supervisor +
pinned workerd + harness + template); a small `stubgate` test server (bundle dir +
static route feed) so the runner is fully testable before M2.
*Accept:* `docker run` runner + stubgate; cold request ≤ 1 s, warm ≤ 5 ms overhead
vs direct workerd; `spike/RESULTS.md` numbers reproduced in-container.

---

## M2 — Control plane + gateway (docs: 04-control-plane.md)

**WP2.1 Gate: store + versions.** SQLite schema + migrations, content-addressed
blob store, multipart ingest → validation (feature matrix in
`proto/features.json`; reject experimental user compat flags, reserved names,
oversize) → immutable version + deployment flip, idempotency by content hash,
audit, token auth (argon2id). Platform API subset per doc.
*Accept:* API tests incl. idempotent re-upload, rollback, and every validation
rejection path.

**WP2.2 Gate: internal API.** `/internal/bundles/:id` (secrets decrypted at build;
marked tmpfs-only), `/internal/routefeed` SSE (snapshot + deltas + Last-Event-ID
resync + prewarm events), internal token auth.
*Accept:* bundle golden tests; route-feed torture test (kill/reconnect/gap →
snapshot resync).

**WP2.3 Gateway.** Route-table client with emptyDir snapshot persistence, host
resolution, `X-Lasso-*` hygiene, rendezvous-hash pod pick via EndpointSlice watch,
DO-worker pinning to the do pool, wall-clock timeouts, retry-once on
`bundle-unavailable`/503-swap-window, WebSocket passthrough, RED metrics.
*Accept:* deploy v2 via gate API → traffic flips < 2 s with zero failed requests
under load (k6 script in repo); **gate down → existing traffic unaffected** (the
critical resilience test).

**WP2.4 Helm chart v0 + e2e.** Chart with gate/gateway/user pool, static
HTTPRoutes, generated secrets; `make e2e`: kind up → helm install → deploy hello
via curl → assert URL serves; teardown.
*Accept:* `make e2e` green in CI.

---

## M3 — CLI v1 (docs: 05-cli.md)

**WP3.1 Core commands.** login/init/deploy/list/versions/rollback/delete/whoami;
wrangler config via wrapped `unstable_readConfig`; esbuild bundling with
wrangler-equivalent rules + `--use-wrangler-build` oracle/escape hatch;
feature-matrix preflight; pretty errors.
*Accept:* fixture projects (JS, TS, wasm, text/json modules, env vars) deploy to
the e2e cluster; `--dry-run --outdir` output matches wrangler's for the fixtures.

**WP3.2 lasso dev + quickstart.** `lasso dev` passthrough to wrangler; README
quickstart rewritten against the real flow.
*Accept:* quickstart followed verbatim on a fresh kind cluster works end to end.

---

## M4 — Data plane: native binding protocol servers (docs: 03-bindings.md; contract: research/static-config-verification.md §1)

**WP4.1 lasso-data KV + secrets.** Service skeleton (auth, scoping headers,
metrics); KV protocol server (get/put/delete/list/bulk-get, metadata-string quirk,
404/410 semantics, expiry); gate KV admin proxy + CLI `kv`; secrets
(encrypt/rotate/redeploy) + CLI `secret`.
*Accept:* protocol conformance tests driven **through a real workerd `kvNamespace`
binding** (not synthetic HTTP), covering every research-§1 detail; CF-docs
semantics suite (types, expiration, metadata, list ordering/cursor).

**WP4.2 R2.** R2 protocol server (CF-R2-Request/Metadata-Size framing, JSON
schemas cross-checked with miniflare's zod schemas, v4code errors); streaming;
multipart; conditionals; content-addressed blobs; CLI `r2`.
*Accept:* conformance through a real `r2Bucket` binding (ranges, etags, multipart,
checksums, list options); 1 GB object streams with bounded memory.

**WP4.3 D1.** `/query`/`/execute` server over per-DB SQLite, CF result envelopes,
array-batch transactions, session-token echo; CLI `d1` incl. migrations.
*Accept:* conformance through the real wrapped binding; batch atomicity; error
shapes; drizzle + kysely smoke tests unmodified.

**WP4.4 Queues.** Producer protocol server (message/batch/metrics, X-Msg-Fmt
handling, v8 bodies opaque); delivery loop (leases, batching, retries, DLQ) →
runner dispatch; CLI `queues`.
*Accept:* e2e producer→consumer with retry + DLQ; batch size/timeout behavior;
crash-mid-lease redelivers; v8-format round-trip fidelity.

---

## M5 — Durable Objects (docs: 03-bindings.md DO section) — *sequential*

**WP5.1 do pool.** DO-mode config rendering (namespaces, uniqueKey management in
gate, localDisk on PVC, enableSql), StatefulSet in chart, gateway pinning,
drain-then-start swap wired end to end.
*Accept:* counter DO e2e: identity stable across user-pool restarts and gateway
restarts; storage + alarms survive do-pod restart; deploy never overlaps processes
(assert via supervisor logs); the workerd chat sample runs.

**WP5.2 DO polish.** Cross-worker restriction enforced at gate with clear errors;
version-skew behavior documented + tested; WebSockets to DOs; `lasso do list`
(namespaces, object counts from disk); backup/restore runbook for DO PVC.
*Accept:* the restriction and skew tests pass; documented runbook verified by a
restore drill in e2e.

---

## M6 — Events + observability (docs: 07-observability.md)

**WP6.1 Cron.** Gate scheduler → dispatch → harness `scheduled()`; outcome
recording; catch-up policy; CLI surfacing.
*Accept:* cron fires within 1 s of schedule; gate restart mid-window honors
policy; failure visible in tail.

**WP6.2 Tail pipeline.** Harness tail batches → gate ingest → ring buffer + live
SSE; `lasso tail` (pretty + json, sourcemaps); sampling caps.
*Accept:* logs + exception stacks correct for a TS fixture; hot-loop worker
doesn't destabilize gate.

**WP6.3 Metrics + dashboard.** Per-doc series everywhere (incl. per-worker
RSS/CPU); ServiceMonitors; Grafana dashboard JSON.
*Accept:* dashboard renders on kind + kube-prometheus-stack; per-worker RED and
memory visible.

---

## M7 — Compat + self-management + hardening

**WP7.1 wrangler-compat API.** `/client/v4` subset (verify, accounts, script PUT,
secrets, subdomain, tail, KV); documented stock-wrangler flow.
*Accept:* stock pinned wrangler `deploy`/`secret put`/`tail`/`kv` work against
lasso; incompatibilities return structured errors.

**WP7.2 Self-management.** Gate k8s module (dedicated pools, drift reconcile,
health-based pod deletion), RBAC in chart, degraded-mode warnings.
*Accept:* namespace marked dedicated → own pool serves it (pods + route flip
asserted); RBAC-off degrades gracefully; compose mode unaffected.

**WP7.3 Security pass.** NetworkPolicies, pod hardening defaults, adversarial
suite (reach internal services from user code, forge scoping headers, hit dispatch
paths unauthenticated, oversized configs/bundles), gVisor pool option, `lasso
audit`.
*Accept:* adversarial suite green; `kube-score`/`polaris` clean.

**WP7.4 Conformance + upgrade automation.** Promote spike + protocol + adversarial
tests into `conformance/` run against {pinned, latest} workerd; bot PR per workerd
release with verdict; divergence runbook. **This suite is the platform's upgrade
safety net — the wire protocols and the dispatch flag sit outside compat-date
guarantees.**
*Accept:* intentionally-broken stub fails the gate; latest real workerd passes or
produces an actionable report.

---

## M8 — Polish to "v1"

Cache API backend (verified protocol, honest LRU), assets v2 (separate store),
`lasso platform backup`, docs site, example gallery (blog, webhook receiver, DO
chat, cron digest, queue pipeline), 24 h soak (memory-flat across deploy churn and
process reaping), versioned release artifacts, SECURITY.md + runbooks.
*Accept:* a stranger with a kind cluster goes from `helm install` to a deployed
worker in < 15 minutes using only the docs.

---

## Post-v1 backlog (unordered)

Web UI · Postgres/S3 backends · mTLS internal · multi-replica do pool (leases +
generation fences) · pool autoscaling · OTel tracing · S3-compat R2 endpoint +
presigned URLs · Python workers (Pyodide cache plumbing; heavy processes — needs
its own sizing) · gradual deployments (weighted versions at the gateway; the
per-version process model makes this natural) · subrequest limits · cross-worker
DO namespaces · import-from-Cloudflare tooling.

## Risk register

| Risk | Likelihood | Mitigation |
| --- | --- | --- |
| Binding wire protocols drift between workerd releases (not compat-date-covered) | Medium | Exact pinning; WP7.4 conformance gate; miniflare's workers are the maintained reference to diff against |
| `service_binding_extra_handlers` (cron/queue dispatch) breaks or changes | Medium (experimental) | Harness-only usage; conformance-gated; documented DO-alarm fallback path (lossy) |
| DO `localDisk` format change (doc-marked experimental) | Medium | Conformance; PVC backups + restore drill (WP5.2); simple one-file-per-object format eases migration |
| Per-worker process memory (~40 MB each) limits density | Low at homelab scale | Measured numbers in spike; lazy spawn + idle reap bound the resident set; dynamic-loading design preserved in git history as the density fallback |
| Cold-start (~30–50 ms + bundle compile) too slow for some workloads | Low | Prewarm on deploy + route hints; keep-warm flag per worker (skip reaping) |
| wrangler unstable APIs churn (CLI) | Medium | Wrapped behind `readProjectConfig()`; `--use-wrangler-build` oracle |
| SQLite contention (gate/data) | Low at homelab scale | WAL + single-writer; Postgres interface reserved |
