# The `lasso` CLI

TypeScript (Node ≥ 22), published as an npm package and as single-file binaries. It
deliberately reuses the wrangler *ecosystem* — config parsing and bundling — while
talking to lasso's platform API. Local development is explicitly delegated:
**`lasso dev` runs `wrangler dev`** (workerd locally via miniflare), because that
experience is already excellent and our runtime is the same engine.

## Principles

1. **A wrangler project deploys unchanged** when it uses supported features:
   `lasso deploy` in a directory with `wrangler.jsonc`/`wrangler.toml` must work with
   zero new files. Platform-specific settings live in optional `lasso.jsonc`
   (platform URL, namespace, pool hints) or env vars (`LASSO_API`, `LASSO_TOKEN`).
2. **Clear feature-gap errors.** If the config uses an unsupported binding, fail
   before upload with the exact list and a docs link (mirrors gate-side validation so
   errors are local and fast).
3. **Thin client.** All truth lives in gate; the CLI is bundler + HTTP + UX. No local
   state beyond auth and caches.

## Command surface (v1)

```
lasso login [--api-url URL]              token setup (interactive or --token)
lasso init [template]                    scaffold (worker + wrangler.jsonc + lasso.jsonc)
lasso dev                                → wrangler dev (passthrough args)
lasso deploy [--ns NS] [--name N] [--no-activate] [--dry-run --outdir DIR]
lasso versions list|view                 immutable history
lasso rollback [VERSION_ID]              repoint deployment (interactive picker)
lasso tail [WORKER] [--format pretty|json]
lasso list                               workers in namespace, URLs, active versions
lasso delete WORKER
lasso secret put|list|delete NAME
lasso kv namespace create|list … / lasso kv key get|put|list|delete …
lasso r2 bucket create|list … / lasso r2 object get|put …
lasso d1 create|list|execute|migrations apply …
lasso queues create|list …
lasso routes add|list|delete PATTERN --worker W
lasso whoami / lasso token create|list|revoke (admin)
lasso platform status                    gate/pools/data health, workerd version
```

## Deploy implementation

1. **Config**: parse with wrangler's exported `unstable_readConfig` (pin wrangler
   version; wrap behind our own `readProjectConfig()` so a vendored parser can
   replace it if the unstable API churns — decision D12). Environments
   (`env.production`) supported the wrangler way.
2. **Bundle**: esbuild with wrangler-equivalent rules (ESM output, `.wasm` as
   CompiledWasm module, `.txt`/`.bin`/`.json` module rules, `nodejs_compat` handling,
   sourcemaps). Escape hatch: `--use-wrangler-build` shells out to
   `wrangler deploy --dry-run --outdir` and ingests its output — guarantees parity
   for exotic projects and doubles as our bundler-conformance oracle in CI.
3. **Upload**: multipart PUT to `/v1/…/versions?activate=true`; print version id,
   URL, and a diff summary (modules changed, bindings added/removed).
4. **Provisioning**: for bindings referencing nonexistent kv/r2/d1/queues, prompt to
   create (or `--yes`), mirroring wrangler's provisioning UX.

## Compatibility matrix (maintained in CLI + docs)

Generated from one source of truth (`proto/features.json`, shared with gate
validation): binding types, wrangler config keys, and their lasso status
(`supported | degraded (note) | unsupported`). `lasso deploy` prints the relevant
subset on first deploy of a project; `lasso platform status --features` prints all.

## Stock-wrangler path

Once gate's `/client/v4` compat layer lands (M7), document the two-line alternative:

```
export CLOUDFLARE_API_BASE_URL=https://api.lasso.lan/client/v4
export CLOUDFLARE_API_TOKEN=<lasso token>
wrangler deploy
```

The custom CLI remains the primary UX (better errors, platform commands, no
account-id weirdness), but this proves the compat surface and gives teams an
off-ramp/on-ramp with zero lock-in.

## Distribution

npm (`npm i -g @lasso/cli` — name TBD with the project name), plus `bun build
--compile` single binaries attached to GitHub releases (macOS arm64/x64, linux
arm64/x64). Homebrew tap post-v1.
