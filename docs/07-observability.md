# Observability

Three signals, all designed around what workerd actually offers (structured logs,
tail workers, the inspector — no native metrics endpoint).

## Logs & tail (the Workers-native signal)

- **Tail workers are the primary channel.** The loader attaches
  `tails: [ctx.exports.TailSink({props:{worker}})]` to every loaded worker. The
  TailSink entrypoint receives structured trace items (console logs, exceptions,
  request outcomes, subrequest summaries) and forwards batches to gate
  `/internal/tail`.
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
| supervisor | workerd RSS/CPU, restarts, watchdog drains, bundle cache hit/miss, loaded-isolate estimate |
| gate | deploys, route-feed clients, envelope builds, cron dispatch outcomes, DB size |
| lasso-data | per-API op counts + latency, queue depths/age, DLQ size, storage bytes per namespace |
| do pool | facet count, alarm fires, storage bytes |

All components ship `/metrics`; the chart includes optional ServiceMonitors and a
starter Grafana dashboard (per-worker RED + platform health). Per-request CPU time is
**not available** (OSS limitation) — wall time and workerd process CPU are what we
have; documented.

## Errors & status

- Request-path errors get stable `X-Lasso-Error` codes (bundle-unavailable,
  worker-timeout, worker-exception, no-route…) and matching gateway metrics; user
  exceptions show up in tail with stacks (sourcemaps applied CLI-side by
  `lasso tail --pretty` using the uploaded sourcemap module).
- `lasso platform status`: gate DB health, pool readiness, data service health,
  workerd version + pinned compat date, route-feed lag.
- Debugging escape hatch (dev clusters only, off by default): supervisor can expose
  workerd's `--inspector-addr` behind a port-forward for Chrome DevTools profiling of
  the loader; never enabled for user pools in normal operation.

## Tracing (post-v1)

OpenTelemetry spans gateway→loader→binding stubs→data with W3C traceparent
propagated on internal hops (the header contract reserves it now). Not in v1; the
tail pipeline covers most homelab debugging needs.
