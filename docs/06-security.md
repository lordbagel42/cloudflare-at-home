# Security model

The honest version, grounded in upstream's own warnings.

## Threat model

**Default posture (homelab): one household, mutually-trusting deployers.** The
platform's job is to stop *accidents* (a buggy worker exhausting memory, SSRF into the
cluster, a leaked secret in logs) and *external attackers* (internet-facing requests),
not to referee hostile tenants against each other.

**Opt-in posture (`isolation: dedicated` per namespace): semi-trusted tenants.**
Process/pod isolation per namespace, optionally gVisor. Even then: we tell users what
Cloudflare tells them — OSS workerd "is not a hardened sandbox"; truly hostile
multi-tenancy needs VM-grade isolation we do not promise.

Non-threats we accept: Spectre-class attacks within a shared pool (Cloudflare's
defenses here are proprietary and fleet-scale); V8 zero-days (mitigated by upgrade
cadence, not prevented); malicious platform admin.

## Trust boundaries (strongest → weakest)

1. **Cluster boundary** — TLS at the k8s Gateway; only gateway and gate API are
   routable from outside.
2. **Pod/process boundary** — a runner pool is one trust domain. Dedicated pools put
   a namespace in its own domain. This is the boundary we *rely on* (cgroups,
   namespaces, seccomp, optional gVisor).
3. **V8 isolate boundary** — separates workers *within* a pool. Real but explicitly
   not hardened (Black Hat 2026: cross-isolate reads in workerd < 1.20260619.1).
   Treated as defense-in-depth, not a guarantee.
4. **Capability boundary** — user `env` contains only loopback stubs scoped by
   props (unforgeable, invisible to user code) + `globalOutbound` pinned to
   public-only network. A worker cannot name, enumerate, or reach anything it wasn't
   given.

## Concrete controls

### Runtime / pods

- workerd pinned ≥ `1.20260619.1` **always** (Check Point fixes); target: track
  latest weekly within one minor week, gated by conformance CI. Renovate-style
  automation opens the bump PR; CI runs conformance; merge = rollout.
- Runner pods: non-root (uid 65532), read-only rootfs, tmpfs for
  config+cache, `seccompProfile: RuntimeDefault`, `capabilities: drop [ALL]`,
  `allowPrivilegeEscalation: false`. Optional `runtimeClassName: gvisor` per pool
  (Helm value; documented perf cost).
- cgroups: pod `memory.max` per pool (default 512 Mi user pool), CPU requests+limits;
  `pids.max` via pod limit.

### Network

- **In-runtime egress control**: user workers' `globalOutbound` = `network(allow =
  ["public"])` — cannot reach RFC1918/ClusterIP even before CNI policy. The loader's
  own outbound is separate. (This is the control Cloudflare implements with a local
  egress proxy; workerd's network service gives it to us in config.)
- **NetworkPolicy** (chart-installed, default on): runners → {lasso-data, do-pool,
  gate:internal, DNS, public internet}; deny all else including pod-to-pod within the
  pool. gate PVC-side services accept only in-cluster peers with the internal token.
- Internal auth: chart-generated bearer token on every internal call; gateway strips
  all `X-Lasso-*` from ingress traffic. mTLS between components is a post-v1 upgrade
  (interface: one TLS config struct per service).

### Secrets & data

- Worker secrets: AES-256-GCM under a master key in a k8s Secret (generated at
  install; rotation command re-encrypts rows). Write-only API. Envelopes carrying
  decrypted secrets: tmpfs-only on runners, never the disk bundle cache.
- Gate DB/blobs, data PVCs: rely on cluster storage encryption (documented
  requirement), plus optional Litestream off-site backup with its own encryption.
- Audit log for every mutating API call (who/what/when), queryable via
  `lasso audit` (admin).

### Platform code

- Loader worker is the highest-value target (trusted JS with `workerLoader` +
  experimental flags). Rules: minimal dependencies (zero runtime npm deps is the
  goal), injected reserved entrypoints namespaced (`__Lasso*`) and rejected in user
  uploads, no privileged fetchers ever placed on user `env`, prototype-pollution
  lint rules, and adversarial tests in conformance (user worker attempts to reach
  internal services, forge props, call reserved entrypoints, exhaust env size).
- Supervisor is the only holder of gate credentials in the pod; loader gets bundles
  only through it (localhost), scoped to ids the gateway actually routed — a
  homelab-strength version of Cloudflare's "sandbox can't enumerate workers" rule.

### Kubernetes self-management RBAC (optional feature, least privilege)

Role limited to lasso's release namespace: get/list/watch pods;
create/patch/delete Deployments **with a required `app.kubernetes.io/managed-by:
lasso` label selector**; patch scale subresources. No cluster-wide anything, no
Secrets access beyond its own. Feature off → RBAC objects not installed.

## Known gaps (documented, not hidden)

| Gap | Status |
| --- | --- |
| Per-isolate CPU/heap limits within a pool | Not possible in OSS workerd. Mitigation: wall-clock timeouts, pool cgroups, watchdog recycling, dedicated pools for noisy tenants. |
| Spectre between workers in one pool | Accepted in shared pools; use dedicated pools + gVisor for stronger stances. |
| Subrequest count limits | Not enforced v1 (loader-side counting is feasible later via the outbound service). |
| Envelope secrets transit as plaintext-within-authed-channel | mTLS post-v1; homelab TLS-everywhere via service mesh is overkill for v1. |
| DO localDisk experimental | Backups + conformance + simple on-disk format (see 03-bindings.md). |
