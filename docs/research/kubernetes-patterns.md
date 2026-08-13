# Research: Kubernetes platform patterns

> Research date: 2026-08-13.
>
> **Framing note:** this report was commissioned when the design was still "operator +
> CRD per Worker." The project has since pivoted: lasso is a self-managing
> *application* deployed onto Kubernetes, and workers are records in the platform's own
> database, not CRDs. The CRD/operator-specific recommendations (§1, §3) are retained
> as background and for the record of why that path was viable; the ingress (§5),
> packaging (§4), scale-to-zero (§7), and Knative (§2) analyses remain directly
> applicable to the pivoted design.

## 1. SpinKube — prior art for the (abandoned) CRD approach

SpinKube (https://www.spinkube.dev/, CNCF Sandbox) runs Fermyon Spin WebAssembly apps
on Kubernetes via an operator; architecturally the closest existing "FaaS runtime as
k8s extension."

Components: **spin-operator** (kubebuilder-based Go operator reconciling `SpinApp` CRs
into Deployments/Services), **containerd-shim-spin** (runwasi-based containerd shim +
RuntimeClass), **runtime-class-manager** (installs shims on nodes), **spin-plugin-kube**
(CLI: scaffold/deploy).

CRDs: `SpinApp` (dev-facing: `image` OCI ref, `executor`, `replicas`,
`enableAutoscaling`, `checks`, `resources`, `variables`, `runtimeConfig`, volumes,
`components`) and `SpinAppExecutor` (platform-facing: `createDeployment`,
`deploymentConfig.runtimeClassName` **or** `deploymentConfig.spinImage` — i.e. run via
node shim or via a plain container image containing the runtime). The two-CRD split
decouples *what* an app is from *how* it runs. Notably SpinKube supports the
plain-container path precisely because node shims are operationally heavy.

Packaging: apps are OCI artifacts pushed via `spin registry push`; scaling via
HPA/KEDA (no scale-to-zero, no activator); the operator creates a ClusterIP Service per
app but **no Ingress/HTTPRoute** — external routing is left to the user.

Refs: https://www.spinkube.dev/docs/topics/architecture/ ·
https://www.spinkube.dev/docs/reference/spin-app/ ·
https://www.spinkube.dev/docs/reference/spin-app-executor/ ·
https://www.spinkube.dev/docs/topics/packaging/

## 2. Knative Serving — what to borrow, what to skip

Architecture (https://knative.dev/docs/serving/): Service → Configuration (every spec
change stamps out a new immutable **Revision**) + Route (traffic splitting across
revisions). Autoscaling: KPA scales on request concurrency and supports scale-to-zero
(default 30 s grace); at zero, the **activator** buffers requests, pokes the
autoscaler, forwards when pods exist. A **queue-proxy** sidecar in every pod measures
concurrency.

**Worth borrowing (applies directly to lasso's control plane):**

- **Immutable revisions + a pointer to "latest"** — cheap (content hash of code+config
  as the version ID), gives instant rollback. Lasso implements this in its own DB
  (versions/deployments), which is even cheaper than CRDs.
- **Concurrency as the scaling signal** for FaaS, if autoscaling is ever added.
- The **Route abstraction** — traffic keyed to version, not process.

**Overkill for a homelab:** the 4-CRD machinery, the activator + queue-proxy data
plane, percentage traffic splitting, and the networking-layer dependency
(Kourier/Contour/Istio).

## 3. Operator implementation practice 2025/2026 (retained for reference)

Default choice was Go + kubebuilder v4.x + controller-runtime v0.24.x
(https://github.com/kubernetes-sigs/kubebuilder). Patterns that still matter to any
k8s-adjacent Go code in lasso (e.g. the self-management module that talks to the k8s
API):

- **Status/conditions conventions** (`metav1.Condition`, `observedGeneration`) — apply
  the same discipline to lasso's own API status objects.
- **Owner references + garbage collection** over finalizers when everything created is
  in-cluster.
- **Server-side apply** with a fixed field manager for any objects lasso manages
  (e.g. scaling a runner Deployment) — avoids fighting other controllers.
- **CEL validation / ValidatingAdmissionPolicy** preferred over admission webhooks;
  zero-webhook designs remove the biggest operational failure mode. (Moot post-pivot:
  no CRDs, no webhooks at all.)

## 4. Packaging as OCI artifacts

OCI registries store arbitrary content: OCI Image/Distribution spec 1.1 (Feb 2024)
added a first-class `artifactType`; **ORAS** (oras.land, `oras-go`) is the standard
client. Spin's approach: config media type holds the app manifest; each Wasm module and
asset is its own layer, so shared layers dedupe. CNCF Wasm WG has a standard layout for
Wasm artifacts.

Trade-offs (unchanged by the pivot, but the pivot changes the *choice*):

| Approach | Pros | Cons |
|---|---|---|
| OCI artifact | Content-addressed digests = free revision IDs; dedup; any registry; normal auth | Needs a registry + a puller |
| ConfigMap | Zero infra | 1 MiB limit, no history, pollutes etcd |
| Inline in CRD | Single object | etcd limits, bloats watches |
| Full container image per worker | No custom pull logic | Image build per deploy = slow inner loop |

**Post-pivot note:** lasso stores bundles in its own control-plane blob store
(content-addressed), so OCI packaging is no longer on the deploy path. ORAS/OCI remains
relevant only as an optional export/import format and for distributing lasso's own
component images.

## 5. Ingress: Gateway API (directly applicable)

**Use Gateway API.** Core resources (GatewayClass, Gateway, HTTPRoute, GRPCRoute,
ReferenceGrant) are GA and production-ready. The decisive external fact:
**ingress-nginx is retired** — retirement announced Nov 11 2025, maintenance ceased
March 2026, repo archived March 24 2026 after repeated CVEs (IngressNightmare
CVE-2025-1974 and four HIGH CVEs in Feb 2026)
(https://www.kubernetes.io/blog/2026/01/29/ingress-nginx-statement/). SIG-Network
shipped ingress2gateway 1.0 (Mar 2026) as the migration path. Building a new platform
on the Ingress API in 2026 is building on a deprecated trajectory. Good homelab Gateway
implementations: Cilium, Traefik, Envoy Gateway, kgateway, Istio.

**Post-pivot application:** lasso's Helm chart attaches the lasso-gateway Service to
the cluster's Gateway via a small number of *static* HTTPRoutes (a wildcard host route
`*.workers.example.lan` + the API host). Per-worker hostnames are resolved *inside*
lasso-gateway, so no per-worker k8s objects are ever created. Custom domains beyond the
wildcard are added either as additional HTTPRoutes (optionally automated by lasso's
self-management module) or by DNS-pointing at the wildcard.

## 6. Hot-reload patterns (mostly superseded)

The four patterns (ConfigMap volume watch; hash-annotation rollout; fetch-on-start init
container; sidecar push) were analyzed for getting new *code* into pods. The pivot to
`workerLoader` dynamic loading makes deploy-time pod churn unnecessary: new code
arrives via the loader's cache-miss callback, keyed by immutable version ids.

Still applicable:

- **Hash-annotation rolling restarts** remain the right mechanism for changes to
  lasso's *own* component config (gateway config, runner static capnp config, workerd
  version bumps) — deterministic, uses stock Deployment machinery.
- ConfigMap volume propagation (kubelet sync period, default ~1 min; `subPath` never
  updates) is why lasso does NOT distribute worker code or route state via ConfigMaps.

## 7. Scale-to-zero (directly applicable)

- **KEDA http-add-on**: scales HTTP workloads to/from zero via an interceptor proxy;
  core KEDA is CNCF Graduated but the HTTP add-on is **still beta** (v0.x) as of
  mid-2026, and its routing overlaps awkwardly with a Gateway API setup.
- **Full Knative**: disproportionate for a homelab.
- **Roll your own activator**: the most complex, bug-prone component class in
  serverless platforms.
- **Don't do it (recommended for v1)**: an idle workerd process with loaded workers
  sits at tens of MB RSS and ~0 CPU. Ten idle workers ≈ noise on any homelab node.
  Ship fixed replicas + optional HPA on the runner pools; keep the gateway→runner
  interface clean so an interceptor could be slotted in later. Bonus post-pivot: since
  lasso-gateway already fronts all traffic and knows per-worker demand, *isolate-level*
  scale-to-zero comes for free (unloaded workers simply aren't resident), which
  removes most of the motivation for pod-level scale-to-zero.

## Sources

https://www.spinkube.dev/docs/topics/architecture/ ·
https://www.spinkube.dev/docs/reference/spin-app/ ·
https://www.spinkube.dev/docs/reference/spin-app-executor/ ·
https://knative.dev/docs/serving/ ·
https://knative.dev/docs/serving/autoscaling/autoscaler-types/ ·
https://github.com/kubernetes-sigs/kubebuilder ·
https://book.kubebuilder.io/reference/good-practices ·
https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/ ·
https://oras.land/docs/ ·
https://github.com/fermyon/spin/blob/main/docs/content/sips/008-using-oci-registries.md ·
https://tag-runtime.cncf.io/wgs/wasm/deliverables/wasm-oci-artifact/ ·
https://www.kubernetes.io/blog/2026/01/29/ingress-nginx-statement/ ·
https://kubernetes.io/blog/2026/03/20/ingress2gateway-1-0-release ·
https://github.com/kedacore/http-add-on ·
https://github.com/stakater/Reloader
