---
title: Staging and preview Workers on Cloudflare
description: "Deploy every non-main branch to one separate preview Worker with its own database, cache bucket and domain, from the same wrangler.jsonc and GitHub Actions workflow."
tags: [cookbook, cloudflare]
topics: [cloudflare, preview, staging, wrangler-environments, github-actions, noindex]
date: 2026-11-03
path: 06-operations/staging-and-preview-workers.md
---
# Staging and preview Workers on Cloudflare

You may want every branch on its own URL. A Next.js app on Workers with OpenNext cannot have that: Cloudflare makes no per-version preview URLs for a Worker that implements a Durable Object, and OpenNext's cache uses two. This recipe is the replacement: one `-preview` Worker that the latest non-main push overwrites, bound to a dev database and its own cache bucket, on a stable `staging.` hostname.

## Prerequisites

- A working production deploy: [../01-start/nextjs-on-workers-opennext.md](../01-start/nextjs-on-workers-opennext.md) and [../01-start/deploy-from-github-actions.md](../01-start/deploy-from-github-actions.md).
- A second Hyperdrive config for the dev database: [../02-data/hyperdrive-supabase-postgres.md](../02-data/hyperdrive-supabase-postgres.md).
- A second R2 bucket for the OpenNext cache.
- The zone on Cloudflare if you want `staging.your-domain.com`: [../04-domains/custom-domain-and-dns-cutover.md](../04-domains/custom-domain-and-dns-cutover.md).
- `wrangler` 4.156.2, `@opennextjs/cloudflare` 1.20.15.

## Steps

1. Create the dev resources: `wrangler r2 bucket create your-app-preview-next-cache` and a Hyperdrive config for the dev database.
2. Add `env.preview` to `wrangler.jsonc`. Wrangler environments do not inherit bindings, so repeat `hyperdrive`, `r2_buckets`, `durable_objects`, `migrations` and `services` (see Code). Point the self-reference service at the preview Worker's name.
3. Give preview a custom domain in `env.preview.routes` and set `workers_dev: true` there. A `routes` entry turns workers.dev off unless you set it.
4. Set `APP_ENV` as a plain var: `"preview"` in `wrangler.jsonc`, overridden with `--var=APP_ENV:production` only in the main-branch deploy step.
5. In the workflow, run `opennextjs-cloudflare deploy` on `main` and `opennextjs-cloudflare deploy -e preview` on every other branch. Cancel superseded previews, never production.
6. Add `X-Robots-Tag: noindex` to every path of a non-production build in `next.config.ts`.
7. Register `staging.your-domain.com` wherever callbacks are allow-listed (OAuth, Stripe test webhook, Supabase redirect URLs).

## Code

### `wrangler.jsonc` (excerpt)
```jsonc
{
  "name": "your-app",
  "main": "worker.ts",
  "routes": [{ "pattern": "your-domain.com", "custom_domain": true }],
  "workers_dev": false,
  "vars": { "APP_ENV": "preview" },
  "hyperdrive": [{ "binding": "HYPERDRIVE", "id": "<HYPERDRIVE_ID>" }],
  "r2_buckets": [
    { "binding": "NEXT_INC_CACHE_R2_BUCKET", "bucket_name": "your-app-next-cache" }
  ],
  "durable_objects": {
    "bindings": [
      { "name": "NEXT_CACHE_DO_QUEUE", "class_name": "DOQueueHandler" },
      { "name": "NEXT_TAG_CACHE_DO_SHARDED", "class_name": "DOShardedTagCache" }
    ]
  },
  "migrations": [{ "tag": "v1", "new_sqlite_classes": ["DOQueueHandler", "DOShardedTagCache"] }],
  "services": [{ "binding": "WORKER_SELF_REFERENCE", "service": "your-app" }],

  // Every non-main branch deploys here (-e preview).
  "env": {
    "preview": {
      "name": "your-app-preview",
      "routes": [{ "pattern": "staging.your-domain.com", "custom_domain": true }],
      "workers_dev": true,
      "vars": { "APP_ENV": "preview" },
      "hyperdrive": [{ "binding": "HYPERDRIVE", "id": "<DEV_HYPERDRIVE_ID>" }],
      "r2_buckets": [
        { "binding": "NEXT_INC_CACHE_R2_BUCKET", "bucket_name": "your-app-preview-next-cache" }
      ],
      "durable_objects": {
        "bindings": [
          { "name": "NEXT_CACHE_DO_QUEUE", "class_name": "DOQueueHandler" },
          { "name": "NEXT_TAG_CACHE_DO_SHARDED", "class_name": "DOShardedTagCache" }
        ]
      },
      "migrations": [{ "tag": "v1", "new_sqlite_classes": ["DOQueueHandler", "DOShardedTagCache"] }],
      "services": [{ "binding": "WORKER_SELF_REFERENCE", "service": "your-app-preview" }]
    }
  }
}
```
Settings such as `placement`, `cache`, `observability` and `upload_source_maps` stay at the top level and apply to both.

### `.github/workflows/cloudflare-deploy.yml` (trigger, concurrency, deploy steps)
```yaml
on:
  push:
    # Branches only: a release tag is not a deploy.
    branches: ["**"]
    paths: ["web/**", ".github/workflows/cloudflare-deploy.yml"]
  workflow_dispatch:

concurrency:
  group: cloudflare-deploy-${{ github.ref }}
  # Never cancel a production deploy mid-flight; superseded previews may go.
  cancel-in-progress: ${{ github.ref != 'refs/heads/main' }}

env:
  APP_ENV: ${{ github.ref == 'refs/heads/main' && 'production' || 'preview' }}
  NEXT_PUBLIC_APP_ENV: ${{ github.ref == 'refs/heads/main' && 'production' || 'preview' }}

# ...install, build (pnpm exec opennextjs-cloudflare build)...

      - name: Deploy production
        if: github.ref == 'refs/heads/main'
        run: pnpm exec opennextjs-cloudflare deploy --var=APP_ENV:production --var=GIT_SHA:"$GIT_SHA"

      - name: Deploy preview
        if: github.ref != 'refs/heads/main'
        run: pnpm exec opennextjs-cloudflare deploy -e preview --var=APP_ENV:preview --var=GIT_SHA:"$GIT_SHA"
```
Secrets handling and the guard steps are in [../01-start/deploy-from-github-actions.md](../01-start/deploy-from-github-actions.md) and [../01-start/secrets-and-env.md](../01-start/secrets-and-env.md).

### `next.config.ts` (excerpt)
```ts
const appEnv = process.env.APP_ENV ?? "preview"

async headers() {
  const noindex = [{ key: "X-Robots-Tag", value: "noindex" }]
  // Preview and staging builds stay out of the index on every host.
  if (appEnv !== "production") return [{ source: "/:path*", headers: noindex }]
  return [
    // ...noindex for session and API paths...
    // The raw platform host serves the same pages as the site's domain.
    {
      source: "/:path*",
      has: [{ type: "host" as const, value: ".*\\.workers\\.dev" }],
      headers: noindex,
    },
  ]
},
```

## Verify

1. Push a branch. The workflow's "Deploy preview" step runs.
2. `curl -sI https://staging.your-domain.com/ | grep -i x-robots-tag` → `x-robots-tag: noindex`
3. `curl -sI https://your-domain.com/ | grep -i x-robots-tag` → no output on a page that should be indexed.
4. `wrangler deployments list --name your-app-preview` shows the latest push.

## Gotchas

- Symptom: preview deploy fails on missing bindings. Cause: wrangler environments do not inherit `hyperdrive`, `r2_buckets`, `durable_objects`, `migrations` or `services`. Fix: repeat each one under `env.preview`.
- Symptom: workers.dev URL stops answering after you add `routes`. Cause: a `routes` entry turns workers.dev off. Fix: `"workers_dev": true` in that environment.
- Symptom: production domain serves a response with `noindex`. Cause: Workers Cache does not key on host, so a noindexed workers.dev response was cached and served on the real domain. Fix: `"workers_dev": false` on the production Worker, and the host-scoped noindex rule above for any case where it is on.
- Symptom: you want a URL per branch. Cause: Cloudflare gives no preview URLs to a Worker with a Durable Object ([docs](https://developers.cloudflare.com/workers/configuration/previews/)). Fix: none today. The last branch pushed wins; two open branches overwrite each other.
- A Worker custom domain cannot be added while a DNS record for that name exists. Delete the `staging` CNAME first.
- A non-production build needs its own `noindex`. Nothing protects a preview Worker by default.
- Preview must not hold live keys. A strict env script refuses a sign-in value equal to its production value and a live Stripe key bound for preview.
