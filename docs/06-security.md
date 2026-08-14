# Security model

The honest version, grounded in upstream's own warnings.

## Threat model

**Default posture (homelab): one household, mutually-trusting deployers.** The
platform's job is to stop *accidents* (a buggy worker exhausting memory, SSRF into the
cluster, a leaked secret in logs) and *external attackers* (internet-facing requests),
not to referee hostile tenants against each other. That said, the process-per-worker
model (D2/D16) means even the default install never relies on V8 isolate boundaries
between workers — every worker is its own OS process with its own bindings config.

**Opt-in posture (`isolation: dedicated` per namespace): semi-trusted tenants.**
A namespace gets its own pool (own pods/cgroups, optionally gVisor). Even then: we
tell users what Cloudflare tells them — OSS workerd "is not a hardened sandbox";
truly hostile multi-tenancy needs VM-grade isolation we do not promise.

Non-threats we accept: Spectre-class attacks within a shared pool (Cloudflare's
defenses here are proprietary and fleet-scale); V8 zero-days (mitigated by upgrade
cadence, not prevented); malicious platform admin.

## Trust boundaries (strongest → weakest)

1. **Cluster boundary** — TLS at the k8s Gateway; only gateway and gate API are
   routable from outside.
2. **Worker process boundary** — every worker version is its own workerd process
   (own address space, own config, own unix socket). This is the boundary we *rely
   on* between workers. The Black Hat 2026 cross-isolate findings (workerd
   < 1.20260619.1) don't cross it — nothing co-resides in a user worker's process
   except the small platform harness worker.
3. **Pod/pool boundary** — cgroups, namespaces, seccomp, optional gVisor; dedicated
   pools give a namespace its own pods and resource budget.
4. **Capability boundary** — a worker's config contains only the bindings it was
   granted, each pointed at lasso-data with platform-injected scoping/auth headers
   the code can never see or alter, plus `globalOutbound` pinned to the public-only
   network service. A worker cannot name, enumerate, or reach anything it wasn't
   given — workerd's capability-based config doing exactly what it was designed for.

## Concrete controls

### Runtime / pods

- workerd pinned ≥ `1.20260619.1` **always** (Check Point fixes); target: track
  latest weekly within one minor week, gated by conformance CI. Renovate-style
  automation opens the bump PR; CI runs conformance; merge = rollout.
- Runner pods: non-root (uid 65532), read-only rootfs, tmpfs for
  config+cache, `seccompProfile: RuntimeDefault`, `capabilities: drop [ALL]`,
  `allowPrivilegeEscalation: false`. Optional `runtimeClassName: gvisor` per pool
  (Helm value; documented perf cost).
- cgroups: pod `memory.max` per pool, CPU requests+limits, `pids.max` via pod limit.
  Per-worker containment inside the pod: `v8Flags` heap cap per process (default
  192 Mi) so a leaking worker aborts alone; supervisor respawns it with backoff.

### Network

- **In-runtime egress control**: user workers' `globalOutbound` = `network(allow =
  ["public"])` — cannot reach RFC1918/ClusterIP even before CNI policy. The harness
  worker's internal-service bindings are separate named capabilities. (This is the control Cloudflare implements with a local
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

- The harness worker is small by design (dispatch relay, tail forwarder, asset
  server) and is the only trusted JS co-resident with user code. Rules: zero runtime
  npm deps, it alone holds the `service_binding_extra_handlers` compat flag and the
  service binding back to internal services; user workers get no binding to the
  harness. Adversarial tests in conformance: user worker attempts to reach internal
  services, hit dispatch paths without supervisor auth, or exhaust process resources.
- The supervisor is the only holder of gate credentials in the pod; it fetches
  bundles itself and renders per-process configs — worker processes receive
  capabilities, never credentials. Configs live on tmpfs mode 0400 per process — a
  homelab-strength version of Cloudflare's "sandbox can't enumerate workers" rule.

### Kubernetes self-management RBAC (optional feature, least privilege)

Role limited to lasso's release namespace: get/list/watch pods;
create/patch/delete Deployments **with a required `app.kubernetes.io/managed-by:
lasso` label selector**; patch scale subresources. No cluster-wide anything, no
Secrets access beyond its own. Feature off → RBAC objects not installed.

## Known gaps (documented, not hidden)

| Gap | Status |
| --- | --- |
| Per-request CPU metering | Not possible in OSS workerd. Mitigation: gateway wall-clock timeouts + per-process heap caps + cgroup CPU shares. (Per-worker *memory* limits are solved by process-per-worker.) |
| Spectre between workers | Largely mitigated by process-per-worker (separate address spaces); residual cross-process/hardware channels are out of scope, as everywhere. |
| Subrequest count limits | Not enforced v1 (counting is feasible later at the egress network service / lasso-data). |
| Secrets transit as plaintext-within-authed-channel | mTLS post-v1; homelab TLS-everywhere via service mesh is overkill for v1. |
| DO localDisk experimental | Backups + conformance + simple on-disk format (see 03-bindings.md). |
| `service_binding_extra_handlers` experimental | Held by the harness worker only; conformance-gated per workerd upgrade; worst-case fallback is HTTP-shim dispatch (a synthetic route on the user worker) with documented semantics loss. |
