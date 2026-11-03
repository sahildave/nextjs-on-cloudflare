---
title: Deploy to Cloudflare Workers from GitHub Actions
description: "A single workflow that builds a Next.js/OpenNext app, deploys main to production, and deploys a hand-started branch run to a separate preview Worker."
tags: [cookbook, cloudflare]
topics: [cloudflare, github-actions, ci, deploy, wrangler, preview-worker]
date: 2026-11-03
path: 01-start/deploy-from-github-actions.md
---
# Deploy to Cloudflare Workers from GitHub Actions

You want a push to `main` to deploy production, and a hand-started run on any branch to deploy that branch somewhere you can open it. This workflow runs `opennextjs-cloudflare deploy` for both: to the production Worker on `main`, and to a separate `-preview` Worker (`-e preview`) on any other branch. It passes `APP_ENV` and the commit SHA to the Worker.

Per-version preview URLs (`wrangler versions upload`, `opennextjs-cloudflare upload`) do not exist for this Worker shape. Cloudflare generates no Version URL for a Worker that implements a Durable Object ([previews docs](https://developers.cloudflare.com/workers/configuration/previews/)), and OpenNext's cache binds two. Branches go to the preview Worker from [../06-operations/staging-and-preview-workers.md](../06-operations/staging-and-preview-workers.md).

## Prerequisites

- An app set up per [./nextjs-on-workers-opennext.md](./nextjs-on-workers-opennext.md).
- An `env.preview` block in `wrangler.jsonc` with its own name, database and cache bucket, per [../06-operations/staging-and-preview-workers.md](../06-operations/staging-and-preview-workers.md). Without it, skip the preview step and run the workflow on `main` only.
- A Cloudflare API token with Workers and R2 edit rights (add D1 if you use it), stored as repo secret `CLOUDFLARE_API_TOKEN`.
- Your Cloudflare account id. It is an id, not a secret, so it can sit in the workflow as `<ACCOUNT_ID>`.
- Build-time env values. Either put them in GitHub secrets, or pull them from a secret manager with the step in [./secrets-and-env.md](./secrets-and-env.md).
- Stay on GitHub Actions rather than Cloudflare Workers Builds if your build prerenders against the live database and needs its secrets. Workers Builds would need those copied into Cloudflare.

## Steps

1. Add the repo secret `CLOUDFLARE_API_TOKEN` (Settings, Secrets and variables, Actions).
2. Create `.github/workflows/cloudflare-deploy.yml` (Code below).
3. If another host still auto-deploys this repo from Git, turn that off so two systems do not both ship (Vercel: [../migrate-from-vercel/README.md](../migrate-from-vercel/README.md)).
4. Push to `main`. The run builds, then `opennextjs-cloudflare deploy --var=APP_ENV:production --var=GIT_SHA:...`.
5. For a preview, start the workflow by hand on a branch (Actions tab, Run workflow). It deploys that branch to the `-preview` Worker, overwriting whatever branch was there. To deploy every branch push instead, widen the `push` trigger as in [../06-operations/staging-and-preview-workers.md](../06-operations/staging-and-preview-workers.md).
6. Read `APP_ENV` and `GIT_SHA` in the app. The same values reach the client bundle as `NEXT_PUBLIC_APP_ENV` and `NEXT_PUBLIC_GIT_SHA`.

How the two modes differ:

| Trigger | Command | Effect |
|---|---|---|
| push to `main` | `opennextjs-cloudflare deploy` | New version at 100% of traffic |
| manual run on a branch | `opennextjs-cloudflare deploy -e preview` | New version of the separate `-preview` Worker; production untouched |

Both populate their own R2 incremental cache.

## Code

### `.github/workflows/cloudflare-deploy.yml`

```yaml
name: cloudflare-deploy

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  id-token: write   # only needed for an OIDC login to a secret manager

concurrency:
  group: cloudflare-deploy-${{ github.ref }}
  # Never cancel a production deploy mid-flight; superseded previews may go.
  cancel-in-progress: ${{ github.ref != 'refs/heads/main' }}

env:
  CLOUDFLARE_ACCOUNT_ID: <ACCOUNT_ID>
  GIT_SHA: ${{ github.sha }}
  NEXT_PUBLIC_GIT_SHA: ${{ github.sha }}
  APP_ENV: ${{ github.ref == 'refs/heads/main' && 'production' || 'preview' }}
  NEXT_PUBLIC_APP_ENV: ${{ github.ref == 'refs/heads/main' && 'production' || 'preview' }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    env:
      CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
    steps:
      - name: Require CLOUDFLARE_API_TOKEN
        run: |
          [ -n "$CLOUDFLARE_API_TOKEN" ] || { echo "::error::Repo secret CLOUDFLARE_API_TOKEN is empty or missing."; exit 1; }

      - uses: actions/checkout@v4

      - uses: pnpm/action-setup@v4
        with:
          version: 10

      - uses: actions/setup-node@v4
        with:
          node-version: 24
          cache: pnpm

      - uses: actions/cache@v4
        with:
          path: .next/cache
          key: next-cache-${{ runner.os }}-${{ hashFiles('pnpm-lock.yaml') }}-${{ hashFiles('src/**') }}
          restore-keys: |
            next-cache-${{ runner.os }}-${{ hashFiles('pnpm-lock.yaml') }}-
            next-cache-${{ runner.os }}-

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      # Build-time env: NEXT_PUBLIC_* plus any server key `next build` reads while
      # prerendering. Add the secret-manager steps from secrets-and-env.md here,
      # or set these as `env:` from GitHub secrets.

      - name: Build
        # next build ran out of heap at Node's default ~2 GB on the runner.
        env:
          NODE_OPTIONS: --max-old-space-size=6144
        run: pnpm exec opennextjs-cloudflare build

      - name: Deploy production
        if: github.ref == 'refs/heads/main'
        run: |
          pnpm exec opennextjs-cloudflare deploy --var=APP_ENV:production --var=GIT_SHA:"$GIT_SHA"
          echo "Deployed $GIT_SHA to production." >> "$GITHUB_STEP_SUMMARY"

      - name: Deploy preview Worker
        if: github.ref != 'refs/heads/main'
        run: |
          pnpm exec opennextjs-cloudflare deploy -e preview --var=APP_ENV:preview --var=GIT_SHA:"$GIT_SHA"
          echo "Deployed $GITHUB_REF_NAME ($GIT_SHA) to the preview Worker." >> "$GITHUB_STEP_SUMMARY"
```

`wrangler.jsonc` keeps `"vars": { "APP_ENV": "preview" }` as the default, so a hand deploy from a laptop cannot claim to be production. Only this workflow passes `production`.

## Verify

```bash
# after a main push
gh run watch                                     # the "Deploy production" step is green
curl -sI https://your-domain.com/ | head -1      # HTTP/2 200
pnpm exec wrangler deployments list              # newest version at 100%

# after a manual branch run
gh run watch                                     # the "Deploy preview Worker" step is green
curl -sI https://staging.your-domain.com/ | head -1   # HTTP/2 200
pnpm exec wrangler deployments list --name your-app-preview   # newest version is this run
```

## Gotchas

- Build killed with exit 137 or "heap out of memory" → Node's default heap (~2 GB) on `ubuntu-latest` → `NODE_OPTIONS: --max-old-space-size=6144` on the Build step only.
- Deploy preview fails on a missing binding → wrangler environments do not inherit bindings → repeat each binding under `env.preview` ([../06-operations/staging-and-preview-workers.md](../06-operations/staging-and-preview-workers.md)).
- Production errors appear from previews → `APP_ENV` defaulted wrong → keep the default `preview` in `wrangler.jsonc` and pass `production` only from the `main` job.
- A secret change shows up as an "unexpected deploy" in the dashboard → every `wrangler secret` write creates a new Worker version → expected; see [./secrets-and-env.md](./secrets-and-env.md).
- The build timed out once while prerendering → `next build` reads the live database → retry before debugging.
- Two production deploys racing → no concurrency control → the `concurrency` block above queues them and never cancels one on `main`.
- A cron trigger added in `wrangler.jsonc` goes live on the next deploy; do not merge it before the domain cutover or the old host's crons run in parallel. See [../05-app-features/cron-triggers.md](../05-app-features/cron-triggers.md).
- App code changes need a full `opennextjs-cloudflare build`; `worker.ts` alone does not. The workflow always builds.
- You want a URL per branch → `opennextjs-cloudflare upload` (`wrangler versions upload`) returns no preview URL for a Worker with a Durable Object ([docs](https://developers.cloudflare.com/workers/configuration/previews/)) → none today; the last branch deployed to the `-preview` Worker wins.
- The deploy fails before checkout with an empty token → the API token secret or the secret manager's identity variable is unset → check both in the first steps and name where to set them, as the "Require" step above does.
- Token scopes that worked (account-scoped): Workers Scripts edit, Workers R2 Storage edit, Hyperdrive read.
- A secret removed from your store stays on the Worker → `--secrets-file` and `secret bulk` only add and update → `wrangler secret delete <KEY>` (`-e preview` for a preview Worker).
