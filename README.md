# Cloudflare at Home

> *"Mom, can we get Cloudflare Workers?" — "No, we have Cloudflare Workers at home."*

A self-hosted Cloudflare Workers platform built on
[**workerd**](https://github.com/cloudflare/workerd) — the open-source runtime that
powers Cloudflare Workers — designed as a **single self-managing application that runs
on top of a Kubernetes cluster**. Deploy workers with a wrangler-style CLI (or stock
wrangler); keep the Workers programming model — fetch handlers, KV, R2, D1, Durable
Objects, cron, queues, service bindings, tail — without the Cloudflare account.

**Status: plan complete, implementation not started.** This repository contains the
full design and an implementation roadmap written so AI coding agents (or humans) can
execute each work package independently. The platform's working codename is **lasso**
(placeholder — see decision D15).

## The design in one paragraph

One Helm install brings up six components: **lasso-gate** (Go control plane: SQLite +
content-addressed bundle store, platform API, wrangler-compatible API, cron scheduler,
auth), **lasso-gateway** (Go ingress proxy: hostname → worker routing from an SSE-fed
route table, timeouts, header hygiene), **runner pools** (pods where a custom Go
**supervisor** manages a fleet of pinned stock **workerd** processes — one process per
active worker version, statically configured, lazily spawned, idle-reaped — mirroring
Cloudflare's production supervisor/runtime split with their isolate lifecycle mapped
onto processes), a **do pool** (workers with Durable Objects, native workerd DO
namespaces with SQLite-on-PVC storage), and **lasso-data** (Go storage backends
speaking workerd's native KV/R2/D1/queue/cache binding protocols). Deploys never touch
the Kubernetes API: the CLI uploads an immutable version to gate, the route feed
announces it, and supervisors spawn the new version's process, flip routing, and drain
the old one — per-worker blue-green, live in under two seconds, no other worker
disturbed. Kubernetes is the substrate; the platform manages itself on top of it
(health, memory watchdogs, process recycling, and — optionally, with RBAC — dedicated
per-tenant runner pools).

## Reading order

| Doc | Contents |
| --- | --- |
| [docs/00-overview.md](docs/00-overview.md) | Goals, non-goals, constraints, glossary, repo layout |
| [docs/01-architecture.md](docs/01-architecture.md) | Components, identity model, data flows, self-management |
| [docs/02-runner.md](docs/02-runner.md) | Supervisor + workerd + loader worker (the core) |
| [docs/03-bindings.md](docs/03-bindings.md) | KV, R2, D1, Durable Objects, queues, cron, secrets, cache, assets |
| [docs/04-control-plane.md](docs/04-control-plane.md) | lasso-gate: data model, APIs, deploy pipeline |
| [docs/05-cli.md](docs/05-cli.md) | The `lasso` CLI |
| [docs/06-security.md](docs/06-security.md) | Honest threat model and hardening |
| [docs/07-observability.md](docs/07-observability.md) | Tail pipeline, metrics, errors |
| [docs/08-kubernetes.md](docs/08-kubernetes.md) | Helm chart, lifecycle, self-management module |
| [docs/09-roadmap.md](docs/09-roadmap.md) | **Milestones M0–M8 as agent-executable work packages** |
| [docs/10-decisions.md](docs/10-decisions.md) | D1–D15: every major decision with rejected alternatives |
| [docs/research/](docs/research/) | Ground-truth research the plan is built on |

## Why these are the load-bearing facts

- workerd ships the entire Workers API surface but **no storage, no limits, no
  control plane** — bindings are client shims pointed at services you provide, and
  lasso-data provides them, speaking the same wire protocols production does.
- Cloudflare's runtime creates isolates lazily, evicts them under pressure, and
  hard-limits them — via proprietary machinery OSS workerd doesn't ship. The faithful
  OSS translation is the **process**: one workerd process per worker version, lazily
  spawned, idle-reaped, heap-capped, replaced on deploy. That's this design, and it's
  also what Cloudflare themselves do when they need enforceable limits.
- workerd is **"not a hardened sandbox"** (their words): trust boundaries are process
  boundaries — which process-per-worker satisfies by construction, even inside the
  shared pool.
- The platform pins workerd exactly and gates every upgrade behind a conformance
  suite, because the binding wire protocols and the two remaining experimental
  dependencies (cron/queue dispatch, DO disk storage) sit outside compat-date
  guarantees.

## What this is not

No edge network, no anycast, no Spectre-grade multi-tenancy, no bug-for-bug Cloudflare
API clone, no workerd fork. Divergences from Cloudflare semantics are documented per
binding, never hidden.

## Getting started (for implementers)

Start at [docs/09-roadmap.md](docs/09-roadmap.md), work package **WP0.1**. Read the
docs listed in each WP before writing code. If reality contradicts the plan, update
the plan in the same PR.
