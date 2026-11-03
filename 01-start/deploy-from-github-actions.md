---
title: Deploy to Cloudflare Workers from GitHub Actions
description: "A single workflow that builds a Next.js/OpenNext app and deploys main to production while any other branch uploads a preview version with its own URL."
tags: [cookbook, cloudflare]
topics: [cloudflare, github-actions, ci, deploy, wrangler, preview-urls]
date: 2026-11-03
path: 01-start/deploy-from-github-actions.md
---
# Deploy to Cloudflare Workers from GitHub Actions

You want a push to `main` to deploy production, and a hand-started run on any branch to give you a preview URL without moving traffic. This workflow does both with `opennextjs-cloudflare deploy` and `opennextjs-cloudflare upload`, and passes `APP_ENV` and the commit SHA to the Worker.

## Prerequisites

- An app set up per [./nextjs-on-workers-opennext.md](./nextjs-on-workers-opennext.md), with `"preview_urls": true` in `wrangler.jsonc`.
- A Cloudflare API token with Workers and R2 edit rights (add D1 if you use it), stored as repo secret `CLOUDFLARE_API_TOKEN`.
- Your Cloudflare account id. It is an id, not a secret, so it can sit in the workflow as `<ACCOUNT_ID>`.
- Build-time env values. Either put them in GitHub secrets, or pull them from a secret manager with the step in [./secrets-and-env.md](./secrets-and-env.md).
- Stay on GitHub Actions rather than Cloudflare Workers Builds if your build prerenders against the live database and needs its secrets. Workers Builds would need those copied into Cloudflare.

## Steps

1. Add the repo secret `CLOUDFLARE_API_TOKEN` (Settings, Secrets and variables, Actions).
2. Create `.github/workflows/cloudflare-deploy.yml` (Code below).
3. If another host still auto-deploys this repo from Git, turn that off so two systems do not both ship (Vercel: [../migrate-from-vercel/README.md](../migrate-from-vercel/README.md)).
4. Push to `main`. The run builds, then `opennextjs-cloudflare deploy --var=APP_ENV:production --var=GIT_SHA:...`.
5. For a preview, start the workflow by hand on a branch (Actions tab, Run workflow). The job summary prints the preview URL.
6. Read `APP_ENV` and `GIT_SHA` in the app. The same values reach the client bundle as `NEXT_PUBLIC_APP_ENV` and `NEXT_PUBLIC_GIT_SHA`.

How the two modes differ:

| Trigger | Command | Effect |
|---|---|---|
| push to `main` | `opennextjs-cloudflare deploy` | New version at 100% of traffic |
| manual run on a branch | `opennextjs-cloudflare upload` | `wrangler versions upload`; a preview URL, no traffic shift |

Both populate the R2 incremental cache.

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

      - name: Upload preview version
        if: github.ref != 'refs/heads/main'
        run: |
          # wrangler appends one JSON object per line here; the version-upload
          # entry carries the preview URL.
          export WRANGLER_OUTPUT_FILE_PATH="$RUNNER_TEMP/wrangler-output.ndjson"
          pnpm exec opennextjs-cloudflare upload --var=APP_ENV:preview --var=GIT_SHA:"$GIT_SHA"
          url="$(jq -r 'select(.type == "version-upload") | .preview_url // empty' "$WRANGLER_OUTPUT_FILE_PATH" | tail -1)"
          [ -n "$url" ] || { echo "::error::wrangler versions upload reported no preview URL. Check that preview URLs are enabled for the Worker."; exit 1; }
          echo "Preview ($GITHUB_REF_NAME, $GIT_SHA): $url" | tee -a "$GITHUB_STEP_SUMMARY"
```

`wrangler.jsonc` keeps `"vars": { "APP_ENV": "preview" }` as the default, so a hand deploy from a laptop cannot claim to be production. Only this workflow passes `production`.

## Verify

```bash
# after a main push
gh run watch                                     # the "Deploy production" step is green
curl -sI https://your-domain.com/ | head -1      # HTTP/2 200
pnpm exec wrangler deployments list              # newest version at 100%

# after a manual branch run
# The job summary line: Preview (<branch>, <sha>): https://<hash>-your-app.<subdomain>.workers.dev
curl -sI "<that url>" | head -1                  # HTTP/2 200
```

## Gotchas

- Build killed with exit 137 or "heap out of memory" → Node's default heap (~2 GB) on `ubuntu-latest` → `NODE_OPTIONS: --max-old-space-size=6144` on the Build step only.
- "wrangler versions upload reported no preview URL" → `preview_urls` is off for the Worker → set `"preview_urls": true` in `wrangler.jsonc`. With `workers_dev: false` (the production setting) the preview URL still works because it is a separate flag.
- Production errors appear from previews → `APP_ENV` defaulted wrong → keep the default `preview` in `wrangler.jsonc` and pass `production` only from the `main` job.
- A secret change shows up as an "unexpected deploy" in the dashboard → every `wrangler secret` write creates a new Worker version → expected; see [./secrets-and-env.md](./secrets-and-env.md).
- The build timed out once while prerendering → `next build` reads the live database → retry before debugging.
- Two production deploys racing → no concurrency control → the `concurrency` block above queues them and never cancels one on `main`.
- A cron trigger added in `wrangler.jsonc` goes live on the next deploy; do not merge it before the domain cutover or the old host's crons run in parallel. See [../05-app-features/cron-triggers.md](../05-app-features/cron-triggers.md).
- App code changes need a full `opennextjs-cloudflare build`; `worker.ts` alone does not. The workflow always builds.
- **Unverified: preview URLs on a Worker with Durable Objects.** OpenNext's cache binds two Durable Objects, and Cloudflare's [previews page](https://developers.cloudflare.com/workers/configuration/previews/) say such a Worker gets no per-version preview URL. If so, the "Upload preview version" step fails its URL check every time. Two production apps ship this step, and neither recorded a successful preview run. Run one manual branch run before you rely on it; if it fails, use a separate preview Worker instead ([../06-operations/staging-and-preview-workers.md](../06-operations/staging-and-preview-workers.md)).
- The deploy fails before checkout with an empty token → the API token secret or the secret manager's identity variable is unset → check both in the first steps and name where to set them, as the "Require" step above does.
- Token scopes that worked (account-scoped): Workers Scripts edit, Workers R2 Storage edit, Hyperdrive read.
- A secret removed from your store stays on the Worker → `--secrets-file` and `secret bulk` only add and update → `wrangler secret delete <KEY>` (`-e preview` for a preview Worker).
