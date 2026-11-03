---
title: "Case study: a Next.js + Payload + Supabase app moved from Vercel to Workers"
description: "The timeline, decisions and trade-offs of moving a Next.js 16 + Payload + Supabase content site from Vercel to Workers in one day, with links to the recipe for each step."
tags: [cookbook, cloudflare]
topics: [cloudflare, workers, opennext, nextjs, payload, supabase, hyperdrive, workers-cache, vercel-migration, case-study]
date: 2026-11-03
path: migrate-from-vercel/case-study-nextjs-payload-supabase.md
---
# Case study: a Next.js + Payload + Supabase app moved from Vercel to Workers

The app is a Next.js 16.3.8 content site with Payload 3 at `/cms`, Supabase Postgres and auth, and Cache Components on. It moved from Vercel to Workers on 3 Nov 2026, when its Vercel usage ran out. This page records what happened and why. The how is in the recipes it links; each one stands alone. Two smaller Next.js apps moved the same day with the same recipes.

## Timeline

1. **Before the move.** Read the plan limits first: on the free plan, cached pages answered 200 and everything else 503 / error 1102. Put the account on Workers Paid. [plans-and-limits](../01-start/plans-and-limits.md)
2. **Adapter and base config** on a feature branch, tested on a `*.workers.dev` Worker. [nextjs-on-workers-opennext](../01-start/nextjs-on-workers-opennext.md)
3. **Database.** A direct TLS connection to Supabase's pooler was dropped, and a shared `pg.Pool` broke across requests. Hyperdrive, a request-scoped pool and pinned placement fixed both. [hyperdrive-supabase-postgres](../02-data/hyperdrive-supabase-postgres.md), [postgres-js-on-workers](../02-data/postgres-js-on-workers.md)
4. **Payload.** `/cms` login failed until PBKDF2 above 100,000 iterations was routed to a JS implementation. [payload-cms-on-workers](../05-app-features/payload-cms-on-workers.md)
5. **Images.** `sharp` replaced with Photon, resvg and jSquash WebP. [images-without-sharp](../05-app-features/images-without-sharp.md)
6. **Auth** kept working; the cookie and cache rules around it changed. [supabase-auth-on-workers](../05-app-features/supabase-auth-on-workers.md)
7. **Caching.** Uncached renders were slower than on Vercel, so public pages moved into Workers Cache, with purge on every write. [workers-cache-edge-caching](../03-caching/workers-cache-edge-caching.md), [cache-invalidation-and-purge](../03-caching/cache-invalidation-and-purge.md)
8. **Deploy and secrets** from GitHub Actions and the team's secret manager; Vercel's auto-deploy went manual-only. The build ran out of heap on `ubuntu-latest` until `NODE_OPTIONS` raised it. [deploy-from-github-actions](../01-start/deploy-from-github-actions.md), [secrets-and-env](../01-start/secrets-and-env.md)
9. **Domain cutover.** The domain moved from Vercel's nameservers to a Cloudflare zone; the zone went active after about 40 minutes. Cron triggers were turned on in the same commit, and the Vercel project was paused at once because the site had no visitors yet. [custom-domain-and-dns-cutover](../04-domains/custom-domain-and-dns-cutover.md), [cron-triggers](../05-app-features/cron-triggers.md), [README](./README.md)
10. **Outage the same day.** The first CMS save expired a tag every shell carried, and every page hung. A rollback did not help. The fix was a backport of opennextjs-cloudflare#1318; after it, a `/cms` publish kept every page at 200 and showed the new tagline at once. [cache-components-ppr-on-workers](../03-caching/cache-components-ppr-on-workers.md), [incident-playbook](../06-operations/incident-playbook.md)
11. **Follow-ups.** Tag-cache Durable Objects pinned to western Europe (158 ms to 16 ms per lookup), a home-page warm after the second purge, logs and error tracking. [observability-and-logs](../06-operations/observability-and-logs.md)

## Decisions

- **OpenNext, not vinext.** Cloudflare recommends vinext for new apps, but its Cache Components support is incomplete (cache profiles, tags, partial shells) and it renders every page per request. OpenNext adapts the real `next build` output and supports `proxy.ts` since 1.20.x. Fallback if both had failed: the existing Dockerfile on Cloudflare Containers.
- **Hyperdrive with caching off,** on the session pooler (`:5432`), origin connection limit 10 against Supavisor's 15. CMS edits must show at once.
- **Worker pinned to `aws:eu-central-1`.** Unpinned, it ran in Mumbai and every query crossed continents. Check `cf-placement: remote-FRA`.
- **Skip the render, do not tune it.** Uncached renders cost P50 223 ms and P90 2 s of CPU. Workers Cache serves public pages without running the Worker.
- **Purge everything on write.** Writes are rare, so one purge-everything beats tracking which pages list which record.
- **GitHub Actions, not Workers Builds.** The build prerenders against the live database and reads its secrets from the secret manager; Workers Builds would need them copied into Cloudflare.
- **Source maps to the error tracker,** not Cloudflare: they are over the 60 MiB upload cap.
- **Nothing that only works on Enterprise.** The Enterprise features on the account were temporary.

## Vercel and Workers compared

Measured from India.

| Area | Vercel | Workers |
|---|---|---|
| Cached page | 1.8 s | 0.27 to 0.4 s; filter prefetches 1 to 8 ms |
| Uncached render | about 1.8 s | 2.7 to 7 s, one 34.6 s outlier; warm misses about 1 to 1.4 s median, fresh-instance misses 3 to 4 s |
| Image processing | `sharp`, native | WASM only; slower, and resvg loads no fonts |
| Database | Direct Postgres works | Needs Hyperdrive, per-request connections, pinned placement |
| Size | No practical limit | 64 MiB uncompressed Worker; source maps do not fit |
| Memory | Up to GBs | 128 MB per isolate |
| Middleware | Native | `proxy.ts` labelled experimental in OpenNext |
| Track record | Default for Next | An earlier OpenNext deployment (July 2026) moved back to Vercel, citing build, runtime and media-path friction |

The cache hides most of the speed gap. The first visitor to an uncached URL after a purge still waits for a full render. Splitting the CMS into its own Worker to cut cold starts is still open.

## Sources

- [OpenNext Cloudflare](https://opennext.js.org/cloudflare), [caching](https://opennext.js.org/cloudflare/caching), [database howto](https://opennext.js.org/cloudflare/howtos/db)
- [Workers Cache](https://developers.cloudflare.com/workers/cache/), [purge](https://developers.cloudflare.com/workers/cache/purge/)
- [Workers limits](https://developers.cloudflare.com/workers/platform/limits/)
- [cloudflare/vinext](https://github.com/cloudflare/vinext)
