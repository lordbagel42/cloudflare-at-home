# Research: the self-hosted workerd ecosystem

> Research date: 2026-08-13. Star counts and activity data approximate as of that date.

## 0. Executive summary

**There is no Kubernetes-native platform for workerd today.** The closest things are a
small Helm chart (union, ~32 stars), a handful of Docker image wrappers, and two
multi-tenant "self-hosted Workers platform" projects (Vorker — effectively abandoned;
wdl — new, active, single-region, not k8s-native). The hard parts this platform needs —
control plane, wrangler-compatible upload API, capnp config generation, storage
backends for KV/R2/D1/DO — all have strong prior art in **Miniflare 3** (a reusable
embedded control plane for workerd, useful as reference source) and in wdl. Cloudflare
explicitly positions workerd as "just the runtime, no platform," and warns it is not a
hardened sandbox for untrusted multi-tenant code (Black Hat 2026 research validated
that warning). So a "Workers on Kubernetes" platform is genuinely unbuilt territory,
but nearly every subcomponent exists somewhere to copy or reference.

## 1. Existing projects that run workerd outside Cloudflare

### workerd itself

- [cloudflare/workerd](https://github.com/cloudflare/workerd) — C++, ~8,600 stars,
  very active. The README explicitly lists **self-hosting Workers apps** as a supported
  use case, recommends **systemd with socket activation** for production, and is
  "unopinionated about hosting environments." Critical caveat, verbatim: **"workerd is
  not a hardened sandbox."** The intro blog post confirms what's *not* included: "the
  full Cloudflare Workers service involves a lot of technology beyond workerd itself,
  including additional security, deployment mechanisms, orchestration, and so much
  more." Cloudflare's suggested deployment model is homogeneous: "every machine runs
  every service."

### Kubernetes-specific

- [willswire/union](https://github.com/willswire/union) — **the only Helm chart** for
  workerd. ~32 stars, 13 commits, low activity. Deploys workerd + cloudflared (tunnel)
  per worker; workers baked into container images via apko/melange. No reconciliation —
  a values.yaml-driven chart. Proof-of-concept quality.
- [pmbanugo/workerd-docker](https://github.com/pmbanugo/workerd-docker) + blog
  ["Running Cloudflare Workers on Docker and Kubernetes"](https://pmbanugo.me/blog/running-cloudflare-workers-on-docker-kubernetes)
  — tutorial-grade example with a Dockerfile and plain k8s Deployment manifests.
- **No community operator/platform for k8s exists.** Targeted searches for "workerd
  operator", "workerd CRD", "workerd custom resource" return nothing. Confirmed gap.

### Multi-tenant "self-hosted Workers platform" projects

- [wdl-dev/wdl](https://github.com/wdl-dev/wdl) — the most serious current attempt.
  "Self-hosted multi-tenant Workers platform on stock Cloudflare workerd with
  multi-replica failover." JavaScript, Apache-2.0, ~34 stars, **created June 2026,
  actively developed**. Uses workerd's **`workerLoader`** to load immutable worker
  versions from Redis/Valkey without restarting workerd; seven microservices (gateway,
  user-runtime, system-runtime, d1-runtime, do-runtime, scheduler, workflows);
  per-resource ownership with lease expiry and generation fences for D1/DO failover;
  supports KV, R2, D1, DO, queues, cron, workflows, assets, service bindings,
  WebSockets; **wrangler-compatible deployment**; Prometheus metrics and log tailing.
  Single-region, not k8s-native. The closest existing thing to this project, minus
  Kubernetes. See research/worker-loader.md §4 for a deep dive.
- [VaalaCat/vorker](https://github.com/VaalaCat/vorker) — Go control plane + web UI +
  multi-node agents over workerd. ~199 stars, but development has moved elsewhere
  (maintenance mode). DO support was in-memory only. Effectively deprecated.
- [giuseppelt/self-workerd](https://github.com/giuseppelt/self-workerd) — "Self-host
  your own FaaS with workerd," ~137 stars, TypeScript, explicitly a proof-of-concept
  (4 commits). Pedagogical reference for the publisher/runtime split.
- [YouXam/workerd-faas](https://github.com/YouXam/workerd-faas) — Oct 2025, ~13 stars,
  prototype. Notable for also using worker-loader and shipping a limited wrangler fork
  (`wrkst`). Dormant.
- [violetbuse/yukako](https://github.com/violetbuse/yukako) — 3 stars, dead since
  ~2024.
- [agentic-research/cloister](https://github.com/agentic-research/cloister) — workerd
  "hypervisor" with declarative capnp manifests, aimed at hosting MCP servers; ~13
  stars, active 2026. Interesting for per-bundle credential scoping.

### Docker images / packaging

- [JacobLinCool/workerd-docker](https://github.com/JacobLinCool/workerd-docker) (~32
  stars, maintained, amd64+arm64) — minimal image running a worker in a container.
- [JacobLinCool/selflare](https://github.com/JacobLinCool/selflare) (~50 stars, updated
  Aug 2026) — the most useful packaging tool: compiles a wrangler project into a
  binary Cap'n Proto workerd config (`selflare compile`) and generates Docker assets,
  with KV, D1, R2, DO, and Cache API working out of the box (reuses Miniflare's
  simulators). The closest existing "wrangler.toml → workerd deployable" tool.
- [Cyb3r-Jak3/docker-workerd](https://github.com/Cyb3r-Jak3/docker-workerd) (~15
  stars), [frafra/workerd-docker](https://github.com/frafra/workerd-docker) (dead) —
  plain image wrappers.
- No official Cloudflare workerd Docker image; everyone downloads the binary from
  GitHub releases or the `workerd` npm package.
- **Nix**: workerd is not packaged in nixpkgs
  ([NixOS/nixpkgs#355460](https://github.com/NixOS/nixpkgs/issues/355460)); the Bazel
  build is the blocker. Plan on wrapping the prebuilt binary.

### Cloudflare's own position

Cloudflare supports self-hosting the *runtime* but has never shipped or promised a
control plane; their answer to "multi-tenant Workers" as a product is **Workers for
Platforms** — paid, Cloudflare-hosted, not open source. The building block they have
opened that matters most: the **dynamic worker loader** binding in workerd, which
removes the old "restart workerd to change config" constraint — the single most
important primitive for a platform on stock workerd.

## 2. Miniflare 3 — reference implementation of a workerd control plane

Source: [`packages/miniflare` in cloudflare/workers-sdk](https://github.com/cloudflare/workers-sdk/tree/main/packages/miniflare).

**How it wraps workerd:** Miniflare is a Node library that translates high-level
`WorkerOptions` into a workerd config, **serializes it to binary Cap'n Proto using
[capnp-es](https://github.com/unjs/capnp-es)** (see `src/runtime/config/index.ts`),
spawns the `workerd` binary with that config on stdin, and manages its lifecycle. Set
`MINIFLARE_WORKERD_CONFIG_DEBUG=1` to dump the generated config. The capnp TypeScript
bindings for `workerd.capnp` are pre-generated in `src/runtime/config/generated/`.

**How the shims work (the key insight):** Since Miniflare 3's rewrite
([PR #656](https://github.com/cloudflare/miniflare/pull/656), "Implement simulators as
Durable Objects"), the simulators are **not Node code** — they are **Workers written in
TypeScript that run inside workerd itself**, implemented as Durable Objects extending
`MiniflareDurableObject` from a `miniflare:shared` module. Storage is workerd's
built-in SQLite for DO storage plus a separate blob store on disk for large values.
See [`src/workers/`](https://github.com/cloudflare/workers-sdk/tree/main/packages/miniflare/src/workers)
— simulators exist for: KV, R2, D1 (SQLite over DO storage), queues, cache,
analytics-engine, assets, dispatch-namespace, workflows, and more. User workers get
ordinary bindings that are actually **service bindings routed to these simulator
workers**.

**Persistence:** ephemeral by default; with `persist` options (what wrangler stores
under `.wrangler/state/v3/`), each service gets an on-disk directory of SQLite DBs +
blobs.

**Relevance to lasso (post-pivot):** we are NOT wrapping miniflare. Miniflare remains
valuable as (a) the authoritative reference for the internal HTTP protocols each
binding shim speaks (KV/R2/D1/queues/cache), (b) a worked example of
simulators-as-workers over DO SQLite storage, and (c) the source of pre-generated capnp
TS bindings if we ever need binary config generation. Its `miniflare:shared` internals
are not a stable public API, its process model is dev-oriented, and its storage is
single-node local disk.

## 3. Wrangler internals relevant to deployment

Docs: [Wrangler API](https://developers.cloudflare.com/workers/wrangler/api/),
[Configuration](https://developers.cloudflare.com/workers/wrangler/configuration/),
[Bundling](https://developers.cloudflare.com/workers/wrangler/bundling/).

**What `wrangler deploy` does:** (1) reads `wrangler.toml`/`wrangler.jsonc` (JSON
schema published; `wrangler types` generates TS types from it); (2) bundles the worker
with **esbuild** (ESM output, module rules for wasm/text/data, node_compat polyfills);
(3) builds a **multipart/form-data PUT to
`/accounts/:account_id/workers/scripts/:script_name`** where one part named `metadata`
is JSON and the rest are module parts; (4) uploads static assets separately if
configured; (5) sets triggers/routes/cron via follow-up API calls.

**Is the upload format documented well enough to clone? Mostly yes.**

- Official docs: [Multipart upload metadata](https://developers.cloudflare.com/workers/configuration/multipart-upload-metadata/)
  — documents `main_module` (the only required field), `compatibility_date`/`_flags`,
  `bindings` (17+ binding types with JSON shapes), DO `migrations`, `annotations`,
  `placement`. Gaps: no full curl walkthrough, thin on part encoding details.
- Module part content types: `application/javascript+module` (esm),
  `application/javascript` (commonjs), `application/wasm`, `text/plain`,
  `application/octet-stream`, `application/source-map`. Community writeups fill gaps,
  e.g. [Deploy Workers Programmatically](https://spiess.dev/note/engineering/infrastructure/deploy-workers-programatically).
- Practical shortcut: **`wrangler deploy --dry-run --outdir dist`** runs config
  resolution + bundling and writes the exact bundle without touching the API. If we
  implement the PUT endpoint plus a handful of GETs, stock `wrangler deploy` can target
  the platform via `CLOUDFLARE_API_BASE_URL` — exactly what **wdl** does. Nobody has
  published a standalone "Cloudflare Workers API emulator" library; wdl's gateway is
  the nearest code to crib.

**`wrangler dev` (local mode) architecture:** bundles with esbuild in watch mode, then
translates config → Miniflare `WorkerOptions`, boots Miniflare (which writes binary
capnp config and spawns workerd), and layers a proxy worker + V8 inspector proxy on top
for hot reload and DevTools. The wrangler.toml→workerd path is production code inside
wrangler.

**Programmatic APIs reusable today:** wrangler exports `unstable_readConfig` (parse
wrangler.toml/jsonc → typed config) and `unstable_getMiniflareWorkerOptions` (config →
Miniflare WorkerOptions). Marked unstable, occasionally buggy on exotic bindings
([#6796](https://github.com/cloudflare/workers-sdk/issues/6796)), but widely used by
third-party tools. `@cloudflare/workers-shared` is *not* useful (Cloudflare-internal
glue workers for Workers Assets).

## 4. Adjacent platforms for design reference

- **SpinKube** ([spinkube.dev](https://www.spinkube.dev/)) — CNCF sandbox; k8s operator
  for Spin Wasm apps. Two CRDs (`SpinApp`, `SpinAppExecutor`), OCI packaging, plain
  Deployments/Services, HPA/KEDA scaling. Was the design template for the pre-pivot
  CRD-based design; still useful for its packaging and executor-decoupling ideas. Key
  divergence: workerd is an ordinary Linux process, so no containerd shim is needed.
- **Supabase edge-runtime**
  ([github.com/supabase/edge-runtime](https://github.com/supabase/edge-runtime)) — the
  closest "vendor open-sourced their FaaS runtime for self-host" analogy. Rust web
  server embedding `deno_core`; a trusted **"main worker" (JS) receives every request
  and programmatically boots/routes to per-function "user workers"** in separate V8
  isolates with memory/CPU/duration limits. That main-worker-as-control-plane pattern
  is exactly analogous to a trusted loader worker + `workerLoader` in workerd. Still
  labeled beta after 3 years — an honest signal of how much work "the last 20%" is.
- **Deno Deploy / Subhosting** — closed API product; not self-hostable. Open pieces:
  **denokv** (self-hostable KV binary, MIT); a newer "deployd" self-host path was
  mentioned on deno.com but unverified in depth.
- **wasmCloud** — CNCF; capability-provider abstraction (KV/blob interfaces with
  swappable backends — conceptually what KV/R2 bindings need), but the component model
  is far from Workers semantics.
- **Knative / OpenFaaS** — container-per-function; useful for revision/traffic
  semantics reference only.

## 5. Multi-tenancy, capnp config generation, wrangler.toml→capnp

**Multi-tenancy guidance:** Cloudflare's position since 2022: "workerd is not, on its
own, a secure way to run possibly-malicious code." Concretely validated by **Check
Point's Black Hat USA 2026 research**
["When Agentic Glue Melts"](https://research.checkpoint.com/2026/when-agentic-glue-melts/):
five memory-corruption vulns in workerd's C++ glue, including an OOB read in URLPattern
letting one worker read another tenant's secrets across the shared heap, and a UAF in
`node:zlib` achieving native code execution (fixed in **workerd v1.20260619.1** — pin
at or above this). Design consequence: **one workerd process per trust domain**,
wrapped in a container/gVisor/microVM boundary; in-process isolates only for workers
within one trust domain. wdl currently accepts the shared-process risk.

**Cap'n Proto config generation libraries:**

- **JS/TS**: [unjs/capnp-es](https://github.com/unjs/capnp-es) (active, what Miniflare
  uses; compiles `workerd.capnp` to TS and writes binary messages — workerd accepts
  binary config via `--binary`). Older capnp-ts is stale. Simplest robust path in JS:
  reuse Miniflare's `serializeConfig` + generated schema, or generate capnp **text**
  format (human-readable; several projects just template it).
- **Go**: [capnproto/go-capnp](https://github.com/capnproto/go-capnp) — mature; can
  compile `workerd.capnp` and emit binary config from Go. No one has published
  workerd-specific Go bindings; we'd be first. Text-format templating from Go is the
  simpler alternative.
- **Rust**: capnproto-rust, with a worked example specifically for workerd:
  [KianNH/capnproto-rust-workerd-configs](https://github.com/KianNH/capnproto-rust-workerd-configs).
- Also: `workerd compile` bundles config+source into one executable — an option for
  immutable per-tenant images.

**wrangler.toml → workerd config:** no polished standalone converter exists. Paths:
(1) wrangler's `unstable_getMiniflareWorkerOptions` + Miniflare's serializer;
(2) [selflare](https://github.com/JacobLinCool/selflare); (3) roll your own against
`workerd.capnp`. A fourth path avoids config generation for the data plane entirely:
run a static trusted-gateway workerd config and load tenant workers dynamically via
**worker-loader** (wdl, workerd-faas, and Cloudflare's own Workers for Platforms all
point this direction). **The pivoted lasso design takes the fourth path.**

## Bottom line

1. Nothing to reinvent at the runtime layer: workerd + the loader-worker pattern is
   proven (wdl, Cloudflare production, Supabase's analogous design).
2. The "platform on Kubernetes" layer is a genuine gap — no prior art to compete with.
3. Speak the Cloudflare API (multipart PUT + metadata JSON) so stock wrangler deploys
   to the platform — documented well enough, and wdl proves feasibility.
4. Take the sandbox warning seriously: process-per-trust-domain + container isolation
   minimum; pin workerd ≥ v1.20260619.1.
