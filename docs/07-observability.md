# Observability

Three signals, all designed around what workerd actually offers (structured logs,
tail workers, the inspector — no native metrics endpoint).

## Logs & tail (the Workers-native signal)

- **Tail workers are the primary channel.** Every per-worker config declares
  `tails = ["harness"]` on the user worker: the harness worker receives structured
  trace items (console logs, exceptions, request outcomes, subrequest summaries) via
  workerd's native tail mechanism and forwards batches to gate `/internal/tail`
  through its internal-service binding.
- Gate keeps a **ring buffer per worker** (SQLite table, capped rows + total bytes,
  default 24 h/50 MB) and fans out live sessions: `lasso tail` ↔ gate SSE; the
  wrangler-compat tail endpoint speaks wrangler's protocol.
- workerd process logs (structuredLogging JSON on stdout) are enriched by the
  supervisor (pool, pod) and go to container stdout — collected by whatever the
  cluster already uses (Loki, etc.). Platform components log JSON to stdout
  uniformly.
- Invocation logs are sampled/capped per worker (defaults: 100 events/s, dropped
  counts surfaced) so a hot loop can't melt gate.

## Metrics (Prometheus everywhere)

| Component | Key series |
| --- | --- |
| gateway | RED per worker: `lasso_requests_total{worker,status}`, latency histograms, in-flight, timeouts, websocket count |
| supervisor | **per-worker-process RSS/CPU** (`lasso_worker_memory_bytes{worker,version}` etc. — a direct benefit of process-per-worker), spawns/reaps/respawns, cold-start latency histogram, drain durations, bundle cache hit/miss |
| gate | deploys, route-feed clients, bundle builds, cron dispatch outcomes, DB size |
| lasso-data | per-API op counts + latency, queue depths/age, DLQ size, storage bytes per namespace |
| do pool | same as supervisor + alarm fires, DO storage bytes |

All components ship `/metrics`; the chart includes optional ServiceMonitors and a
starter Grafana dashboard (per-worker RED + memory + platform health). Per-*request*
CPU time is **not available** (OSS limitation) — but per-*worker* CPU/memory now is,
via procfs on each child process; documented.

## Errors & status

- Request-path errors get stable `X-Lasso-Error` codes (bundle-unavailable,
  worker-timeout, worker-exception, no-route…) and matching gateway metrics; user
  exceptions show up in tail with stacks (sourcemaps applied CLI-side by
  `lasso tail --pretty` using the uploaded sourcemap module).
- `lasso platform status`: gate DB health, pool readiness, data service health,
  workerd version + pinned compat date, route-feed lag.
- Debugging escape hatch (dev clusters only, off by default): the supervisor can
  start a specific worker's process with `--inspector-addr` behind a port-forward for
  Chrome DevTools profiling — per-worker, on demand (`lasso debug <worker>`), another
  direct benefit of process-per-worker; never enabled in normal operation.

## Tracing (post-v1)

OpenTelemetry spans gateway→supervisor→worker→data with W3C traceparent
propagated on internal hops (the header contract reserves it now). Not in v1; the
tail pipeline covers most homelab debugging needs.
