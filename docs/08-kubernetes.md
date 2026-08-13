# Running on Kubernetes: chart, lifecycle, self-management

lasso is *an application on* Kubernetes: everything below is about the platform's own
pods, never per-worker objects.

## Helm chart (`deploy/helm/lasso`)

```yaml
# values.yaml (shape)
baseDomain: workers.home.example        # wildcard host: *.{baseDomain}
apiDomain: api.home.example
gateway:
  replicas: 2
  timeouts: { defaultWallMs: 30000 }
gate:
  storage: { size: 10Gi, className: "" }
  backup: { litestream: { enabled: false, url: "" } }
pools:
  user: { replicas: 2, memory: 512Mi, cpu: "1", gvisor: false }
do:
  storage: { size: 5Gi }
data:
  storage: { size: 20Gi }
ingress:
  gatewayRef: { name: main-gateway, namespace: infra }   # existing Gateway API gateway
  createHTTPRoutes: true
selfManagement:
  rbac: true            # dedicated pools + pool scaling
workerd:
  version: 1.20260813.1  # informational; binaries are baked into images tagged in lockstep
```

Installed objects: Deployments (gate, gateway, user pool, data), StatefulSets (do
pool), Services, PVCs, two static **HTTPRoutes** (`*.{baseDomain}` → gateway;
`{apiDomain}` → gate), NetworkPolicies, Secrets (master key, internal token,
bootstrap admin token — generated once via pre-install hook), optional
RBAC/ServiceAccount for self-management, optional ServiceMonitors.

TLS: assumed at the referenced Gateway (cert-manager wildcard cert — documented
recipe, not managed by lasso).

Also shipped: a `docker-compose.dev.yaml` running the same images flat (no k8s) for
development and as proof the platform degrades gracefully without the k8s API.

## Pod lifecycle & upgrades

- **Gateways/gate/data**: standard rolling updates; gateways carry a persisted route
  snapshot (emptyDir) so cold starts don't depend on gate being up.
- **Runner pools**: rolling with `maxUnavailable: 0, maxSurge: 1`; preStop = drain
  (readiness off → gateway stops routing within its health interval → wait for
  in-flight ≤ 0 or grace timeout). Loaded isolates are cache — nothing to migrate.
- **do pool**: single replica v1; update = brief DO unavailability window
  (`Recreate`-style, documented). In-flight object state is on the PVC; alarms
  catch up on start. v2's lease design removes the window.
- **workerd upgrades**: new runner image (CI-built per upstream release, conformance
  suite must pass) → `helm upgrade` → rolling restart. Compat dates guarantee user
  code; conformance guarantees platform code.
- **Backups**: gate PVC (SQLite + blobs) via Litestream or scheduled
  `sqlite3 .backup` CronJob (chart-optional); data + DO PVCs via VolumeSnapshots
  (documented, cluster-dependent). `lasso platform backup` triggers a consistent
  gate backup on demand.

## Self-management module (in gate, behind `selfManagement.rbac`)

Narrow client of the k8s API (in-cluster config), all objects labeled
`app.kubernetes.io/managed-by: lasso`:

1. **Dedicated pools**: namespace marked `isolation: dedicated` → create Deployment
   `lasso-pool-<ns>` (same runner image/config, pool env = ns), Service, and route
   the namespace there via the route feed. Deleting the marker drains and removes.
2. **Pool scaling (v1.5)**: scale pool Deployments between min/max on gateway
   concurrency metrics (simple target-tracking; no activator, floor ≥ 1).
3. **Health actions**: a pod failing loader health repeatedly gets deleted (kubelet
   reschedules) — after supervisor-local recovery has already been tried.
4. **Drift**: on boot and periodically, reconcile expected pools vs live Deployments
   (recreate missing, adopt labeled strays, never touch unlabeled objects).

Without RBAC, these degrade to warnings; the chart's static topology carries all
core function.

## Resource footprint (target)

Idle platform (2 gateway, 1 gate, 2 user runners, 1 do, 1 data):
≈ 6 pods, ~400–700 MB RSS total, negligible CPU. Fits a single-node homelab
comfortably; nothing requires multi-node, and nothing breaks with it except do-pool
PVC affinity (RWO — StatefulSet handles it).
