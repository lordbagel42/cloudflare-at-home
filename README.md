# Cloudflare at Home

> *"Mom, can we get Cloudflare Workers?" — "No, we have Cloudflare Workers at home."*

A plan for a self-hosted Cloudflare Workers platform built on
[**workerd**](https://github.com/cloudflare/workerd) — the open-source runtime that
powers Cloudflare Workers — designed as a single self-managing application that runs on
top of a Kubernetes cluster.

**Status: planning — research in progress.** This repository will contain the full
design and an implementation roadmap written so that AI coding agents (or humans) can
execute each milestone independently.

## Design direction (settled so far)

- **An application on Kubernetes, not a Kubernetes extension.** One Helm install brings
  up the whole platform: a custom control plane with its own API and storage, a gateway,
  a pool of runner pods, and storage-backend services. Deploying a worker talks to the
  platform's API — not the Kubernetes API. No per-worker CRDs or kubectl involvement;
  Kubernetes is the substrate (Deployments, Services, PVCs, HPA), and the platform
  manages itself on top of it.
- **A custom supervisor wrapping stock workerd** — not miniflare. Runner pods run
  long-lived workerd processes that hot-load worker code via workerd's dynamic worker
  loading (`workerLoader`), modeled on how Cloudflare's own production stack wraps the
  runtime. Miniflare serves as reference source code only.
- **Wrangler-style developer experience.** A custom CLI that reads `wrangler.jsonc`,
  bundles with esbuild, and deploys to the platform API; local dev remains
  `wrangler dev`. A wrangler-compatible API surface is planned so stock wrangler can
  also target the platform.
- **The Workers programming model, honestly scoped.** KV, R2, D1, Durable Objects,
  cron, queues, and service bindings backed by platform-provided services over cluster
  storage — with no pretense of Cloudflare's edge network or Spectre-grade multi-tenant
  isolation.

Full architecture docs, component designs, security model, and the milestone roadmap
land here once the remaining research (Cloudflare's production runtime architecture and
workerd's dynamic loading internals) is synthesized.
