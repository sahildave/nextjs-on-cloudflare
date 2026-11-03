---
title: Workers plans and limits
description: "What Workers Paid is needed for, and the limits that shaped a Next.js + Payload deploy: size, CPU, memory, startup time, cron."
tags: [cookbook, cloudflare]
topics: [cloudflare, workers, plans, limits, pricing]
date: 2026-11-03
path: 01-start/plans-and-limits.md
---
# Workers plans and limits

Read this before you move an app, because a plan limit looks like a code bug. It lists what needs the paid plan and the limits that matter for a full-stack Next.js app.

## Prerequisites

- A Cloudflare account.
- Limits change. Check the current [Workers limits page](https://developers.cloudflare.com/workers/platform/limits/) before relying on a number here (checked 3 Nov 2026).

## Steps

1. Put the account on Workers Paid before the first real deploy. Free plan symptoms are in Gotchas.
2. Check the limits table against your bundle: run `wrangler deploy --dry-run --outdir bundled/` and read "Total Upload".
3. Design for the plan you will keep paying for. A feature you only have on a trial or a temporary Enterprise tier stops working when that ends.

## Limits that mattered

| Limit | Value (Workers Paid) | What it meant in practice |
|---|---|---|
| Worker size | 64 MiB uncompressed, no compressed cap | Next + Payload fits. Source maps do not (60 MiB upload cap), so map upload is off. OpenNext's docs still say 10 MiB compressed; the limits page is newer. |
| Startup | 1 s to evaluate the bundle | A large Payload + Next bundle can miss it. Preview does not enforce it, so deploy once to `workers.dev` and read startup time. |
| Memory | 128 MB per isolate | Rules out sharp-style image work and big in-memory buffers. |
| CPU per request | 30 s default, up to 5 min via `limits.cpu_ms` | A Payload login hashing at 600,000 PBKDF2 iterations in JS costs about 15 s, so set `cpu_ms: 300000`. |
| CPU, free plan | 10 ms per request | A fresh Next + Payload render needs tens to hundreds of ms. Error 1102. |
| Cron CPU | 30 s if more often than hourly, 15 min if hourly or rarer | Fine for hourly/daily/weekly jobs. |
| Cron triggers, free plan | 5 per account | Deploy silently skips the rest. |
| Subrequests and connections | per-request limits apply | Use Hyperdrive and one connection per query; see [../02-data/hyperdrive-supabase-postgres.md](../02-data/hyperdrive-supabase-postgres.md). |
| Cloudflare for SaaS custom hostnames | 100 included on a Free, Pro or Business zone, then $0.10 each a month, up to 50,000; apex proxying is Enterprise | See [../04-domains/cloudflare-for-saas-custom-hostnames.md](../04-domains/cloudflare-for-saas-custom-hostnames.md). |
| PBKDF2 iterations | capped at 100,000 in production runtime | Local workerd has no cap; only `wrangler dev --remote` shows it. See [../05-app-features/payload-cms-on-workers.md](../05-app-features/payload-cms-on-workers.md). |


## Code

Nothing to copy. The one config line plan limits touch:

```jsonc
// wrangler.jsonc
"limits": { "cpu_ms": 300000 }   // Paid plan only; the free plan ignores it and stays at 10 ms
```

## Verify

```bash
pnpm exec wrangler deploy --dry-run --outdir bundled/
# Total Upload: <size> MiB   (must be under 64 MiB uncompressed)

curl -s -o /dev/null -w "%{http_code}\n" https://your-app.<subdomain>.workers.dev/some-uncached-page
# 200 on Paid; 503 with error code 1102 on Free when the render exceeds 10 ms CPU
```

In the dashboard, Workers & Pages, your Worker, Metrics shows CPU time, startup time and errors by status.

## Gotchas

- Cached pages return 200 while everything else returns 503 / 1102 → free plan CPU cap → upgrade, then recheck before debugging code.
- Cron jobs missing after deploy → free plan's 5-trigger account limit → Paid, or consolidate. See [../05-app-features/cron-triggers.md](../05-app-features/cron-triggers.md).
- Deploy fails on startup in production but preview works → 1 s startup limit is not enforced locally → deploy to `workers.dev` and watch startup time; trim the bundle or split out the CMS into its own Worker.
- Source maps upload fails → over 60 MiB → turn upload off and ship maps to your error tracker.
- Enterprise features (bot management, Logpush) work, then stop → they came with a temporary tier → design for Free/Pro, treat Enterprise as a bonus.
- Cold starts: a fresh instance loads a ~50 MB bundle and boots Payload, so 3 to 4 s misses on a cold instance. Cache to avoid it ([../03-caching/workers-cache-edge-caching.md](../03-caching/workers-cache-edge-caching.md)); do not tune the render.
