# Roadmap: milestones and work packages

Written to be executed by AI coding agents (Opus/Sonnet tier) or humans, one work
package (WP) at a time. Rules for executors:

- **Read first**: the WP's listed docs + research docs. They contain the settled
  decisions; don't re-litigate them in code. If reality contradicts the plan (an API
  changed upstream, a flag moved), fix the docs in the same PR and say so.
- **Every WP ends green**: code + tests + a runnable demo command in the WP's
  acceptance section. CI must pass. No WP leaves the repo in a state where `make e2e`
  regresses.
- **Pin everything** (workerd version, wrangler version, Go/Node toolchains) in one
  place: `versions.lock.json` at repo root, created in M0.
- Conventions: Go 1.26+, `golangci-lint`; TypeScript strict, Node 22, pnpm; conventional
  commits; every service gets a `Dockerfile` + `make run-local`.

Milestones are sequential; WPs inside a milestone are parallelizable unless noted.

---

## M0 — Repo foundation

**WP0.1 Scaffolding.** Monorepo layout per 00-overview.md; Makefile; Go workspace;
pnpm workspace; `versions.lock.json` (workerd `1.20260813.1` or newest passing,
wrangler, toolchains); Apache-2.0 LICENSE; CI (GitHub Actions): lint + unit + (stub)
e2e jobs; kind-based dev cluster script (`hack/kind-up.sh`).
*Accept:* CI green on empty packages; `hack/kind-up.sh` yields a cluster with a
Gateway API implementation (Envoy Gateway) installed.

**WP0.2 workerd spike (de-risk, throwaway allowed).** A `spike/` program: Go spawns
pinned workerd with a hand-written text capnp config containing a minimal loader
worker with a `workerLoader` binding; loader loads a hello worker from an inline
envelope on first request; second worker isolated (env separation, blocked
globalOutbound); `abortIsolate` eviction verified; `stub.scheduled()` +
`stub.queue()` verified under `service_binding_extra_handlers`.
*Accept:* `go test ./spike/...` proves: dynamic load, isolation, eviction, scheduled
+ queue dispatch, on the pinned workerd. **This test set is the seed of the
conformance suite.** Findings that contradict docs/research → update docs.

---

## M1 — Runner (docs: 02-runner.md)

**WP1.1 Supervisor.** Config template rendering, socket-FD spawn, control-fd
readiness, crash respawn with backoff, RSS watchdog (drain/restart), health+metrics
endpoints, localhost bundle-cache server (disk cache + gate fetch with token,
hash-verified).
*Accept:* unit tests with a fake workerd + real-workerd integration test: kill -9 the
child mid-request → next request succeeds after respawn; watchdog triggers on
synthetic RSS; bundle cache serves offline once warm.

**WP1.2 Loader worker.** TS project (zero runtime deps target): request path
(header parse, getWorker, entrypoint fetch), bundle envelope parsing, WorkerCode
assembly for all module types, sibling-version eviction, `/__lasso/health`,
`/__lasso/dispatch/*`, TailSink stub (log to console for now). Built with esbuild
into the runner image.
*Accept:* wd-test/vitest-style tests against real workerd (reuse spike harness): two
workers, two versions, correct isolation, eviction on activation, dispatch outcomes
round-trip.

**WP1.3 Runner image + stub gate.** Dockerfile (distroless, multi-arch), a 200-line
`stubgate` test server (serves envelopes from a directory, static route feed) so the
runner is testable without M2.
*Accept:* `docker run` runner + stubgate; `curl -H 'X-Lasso-Worker: default/hello@v1'`
returns the worker's response in < 5 ms warm, < 300 ms cold.

---

## M2 — Control plane + gateway (docs: 04-control-plane.md)

**WP2.1 Gate: store + versions.** SQLite schema + migrations, blob store, multipart
ingest → validation (feature matrix from `proto/features.json`) → immutable version +
deployment flip, idempotency by content hash, audit. Platform API subset: versions,
deployments, workers, namespaces, tokens (argon2id), healthz.
*Accept:* API tests incl. idempotent re-upload, rollback, validation rejections
(unsupported binding, oversize, bad compat date, reserved names).

**WP2.2 Gate: internal API.** `/internal/bundles/:id` (envelope build, secrets
decrypt, caching), `/internal/routefeed` SSE (snapshot + deltas + Last-Event-ID
resync), internal token auth.
*Accept:* envelope golden tests; route feed torture test (kill/reconnect/large gap
→ snapshot resync).

**WP2.3 Gateway.** Route table client (SSE + emptyDir snapshot persistence), host
resolution, header hygiene (strip `X-Lasso-*`), rendezvous-hash pod pick via
EndpointSlice watch (fallback: Service DNS), wall-clock timeouts, retry-once on
bundle-unavailable, WebSocket passthrough, RED metrics.
*Accept:* integration: deploy v2 via gate API → traffic flips < 2 s with zero
failed requests under load (vegeta/k6 script in repo); gate down → traffic
unaffected (the M-critical resilience test).

**WP2.4 Helm chart v0 + e2e.** Chart with gate/gateway/user-pool, static routes, and secrets
generation; `make e2e`: kind up → helm install → deploy hello via curl → assert URL
serves; teardown.
*Accept:* `make e2e` green in CI.

---

## M3 — CLI v1 (docs: 05-cli.md)

**WP3.1 Core commands.** login/init/deploy/list/versions/rollback/delete/whoami;
wrangler config via wrapped `unstable_readConfig`; esbuild bundling with wrangler
rules + `--use-wrangler-build` escape hatch; feature-matrix preflight; pretty errors.
*Accept:* fixture projects (plain JS, TS, wasm, text/json modules, env vars) deploy
to the e2e cluster; `--dry-run --outdir` output matches wrangler's for the fixtures
(oracle test).

**WP3.2 lasso dev + docs site seed.** `lasso dev` passthrough to wrangler; README
quickstart rewritten against the real flow; per-command help.
*Accept:* quickstart followed verbatim on a fresh kind cluster works end to end.

---

## M4 — Data plane bindings (docs: 03-bindings.md)

**WP4.1 lasso-data KV + secrets.** Service skeleton (auth, metrics), KV storage +
REST, loader KV entrypoint class, gate KV admin proxy + CLI `kv` commands; secret
storage in gate (encrypt/rotate) + envelope wiring + CLI `secret`.
*Accept:* KV conformance tests ported from CF docs semantics (types, expiration,
metadata, list ordering/cursor); secret round-trip; `wrangler kv` shapes match.

**WP4.2 R2.** Metadata store + content-addressed blobs, streaming, multipart,
conditional ops; loader R2 class; CLI `r2`.
*Accept:* R2 semantics tests (ranges, etags, multipart, checksums, list
delimiters); 1 GB object streams with bounded loader memory.

**WP4.3 D1.** Per-DB SQLite exec service, CF-shaped result envelopes, batch
transactions; loader D1 class; CLI `d1` incl. migrations.
*Accept:* D1 test suite incl. batch atomicity, error shapes, `meta` fields;
drizzle/kysely smoke tests run unmodified.

**WP4.4 Queues.** Storage (leases, retries, DLQ), delivery loop → runner dispatch,
producer class; CLI `queues`; wrangler config consumer settings honored.
*Accept:* e2e producer→consumer with retry + DLQ; batch timeout/size behavior;
crash-mid-lease redelivers.

---

## M5 — Durable Objects (docs: 03-bindings.md DO section) — *the hard one, sequential*

**WP5.1 do pool.** Runner DO-mode config (supervisor DO namespace, localDisk on PVC,
enableSql), StatefulSet in chart, gateway/loader routing of DO calls.
*Accept:* counter DO e2e: identity stable across user-pool pod restarts; storage
survives do-pod restart; two workers sharing a DO namespace see the same objects.

**WP5.2 Facet lifecycle + bindings.** DONamespace loader class (id derivation, CF id
format), version-skew rule, facet abort on activation (tunable), alarms, WebSockets
to DOs.
*Accept:* alarm fires after do-pod restart; version activation behavior matches the
documented rule; chat-room sample (from workerd samples) runs.

---

## M6 — Events + observability (docs: 07-observability.md)

**WP6.1 Cron.** Gate scheduler, dispatch, outcome recording, catch-up policy; CLI
surfacing.
*Accept:* cron fires within 1 s of schedule under normal load; gate restart
mid-window honors policy.

**WP6.2 Tail pipeline.** TailSink → gate ingest → ring buffer + live SSE;
`lasso tail` (pretty + json, sourcemap application); sampling caps.
*Accept:* `lasso tail` shows logs+exceptions with correct stacks for a TS fixture;
hot-loop worker doesn't destabilize gate (cap kicks in).

**WP6.3 Metrics + dashboard.** All components' metrics per doc; ServiceMonitors;
Grafana dashboard JSON in repo.
*Accept:* dashboard renders on kind + kube-prometheus-stack; per-worker RED visible.

---

## M7 — Compat + self-management + hardening

**WP7.1 wrangler-compat API.** `/client/v4` translation subset (verify, accounts,
script PUT, secrets, subdomain, tail, KV); documented stock-wrangler flow.
*Accept:* stock pinned wrangler `deploy`, `secret put`, `tail`, `kv` work against
lasso e2e; incompatibilities return structured errors.

**WP7.2 Self-management.** Gate k8s module (dedicated pools, drift reconcile,
health-based pod deletion), RBAC in chart, degraded-mode warnings.
*Accept:* mark namespace dedicated → new pool serves it (e2e asserts pod set + route
flip + old pool stops seeing traffic); RBAC-off degrades gracefully.

**WP7.3 Security pass.** NetworkPolicies, pod hardening defaults, adversarial test
suite (06-security.md list: reach internal services, forge props, reserved
entrypoints, env bombs), gVisor pool option, audit command.
*Accept:* adversarial suite green; `kube-score`/`polaris` clean on chart output.

**WP7.4 Conformance + upgrade automation.** Promote spike/adversarial/binding tests
into `conformance/` run against a matrix {pinned workerd, latest workerd}; bot PR on
new workerd release with conformance verdict; runbook for divergence.
*Accept:* intentionally-broken workerd stub fails the gate; latest real workerd
passes or produces an actionable report.

---

## M8 — Polish to "v1"

Cache API backend (honest LRU), assets v2 (separate store), `lasso platform backup`,
docs site, example gallery (blog, webhook receiver, DO chat, cron digest, queue
pipeline), load/soak test (24 h, memory-flat), versioned release artifacts (images +
chart + CLI binaries), SECURITY.md + upgrade/backup runbooks.
*Accept:* a stranger with a kind cluster gets from `helm install` to a deployed
worker in < 15 minutes using only the docs.

---

## Post-v1 backlog (unordered)

Web UI dashboard · Postgres/S3 backends · mTLS internal · multi-replica do pool
(leases + generation fences, wdl-style) · pool autoscaling · OTel tracing · S3-compat
R2 endpoint + presigned URLs · Python workers (Pyodide cache plumbing) · gradual
deployments (weighted versions at gateway) · subrequest limits · Workers Analytics
Engine shim · import from Cloudflare (pull scripts/KV via CF API).

## Risk register (watch these)

| Risk | Likelihood | Mitigation |
| --- | --- | --- |
| `workerLoader`/`service_binding_extra_handlers` breaking change upstream | Medium (experimental) | Exact pinning; conformance gate; thin abstraction (`runner/loader/loaderapi.ts`) so a strategy swap (e.g. config-per-worker + process pools) stays contained |
| DO `localDisk` format change | Medium | Same conformance gate; backup docs; simple format eases migration |
| wrangler unstable APIs churn (CLI) | Medium | Wrapped behind `readProjectConfig()`; `--use-wrangler-build` oracle keeps parity checkable |
| Loader JS perf under high fan-out | Low | It's the same architecture Cloudflare/wdl run; benchmark in M8 soak |
| SQLite write contention (gate/data) | Low at homelab scale | WAL + single-writer discipline; Postgres interface reserved |
