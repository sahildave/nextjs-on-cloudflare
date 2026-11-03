---
title: Migrate a Next.js app from Vercel to Cloudflare Workers
description: "The Vercel-specific half of a move to Workers: cutover order, DNS and nameservers off Vercel, env var and deploy changes, vercel.json crons, firewall rules, what gets worse, and the stale-DNS 503 after pausing the Vercel project."
tags: [cookbook, cloudflare]
topics: [cloudflare, vercel-migration, dns, nameservers, cron, env, firewall, cutover]
date: 2026-11-03
path: migrate-from-vercel/README.md
---
# Migrate a Next.js app from Vercel to Cloudflare Workers

The general recipes in this cookbook set up a Next.js app on Workers from scratch. This file holds only what changes because the app is coming from Vercel. Read it alongside them; each step names the general recipe that does the work. A worked example with timings is in [case-study-nextjs-payload-supabase.md](./case-study-nextjs-payload-supabase.md).

## Prerequisites

- The app builds and serves on a `*.workers.dev` Worker: [nextjs-on-workers-opennext.md](../01-start/nextjs-on-workers-opennext.md).
- Workers Paid on the Cloudflare account: [plans-and-limits.md](../01-start/plans-and-limits.md).
- The Vercel CLI logged into the team that owns the project and the domain.

## Order of cutover

1. Port the app and test it on `workers.dev`. Leave Vercel serving production.
2. Move deploys to GitHub Actions ([deploy-from-github-actions.md](../01-start/deploy-from-github-actions.md)) and turn off Vercel's Git deploys (below), so two systems do not both ship.
3. Move env vars (below, and [secrets-and-env.md](../01-start/secrets-and-env.md)).
4. Write the Worker's cron triggers but leave them off ([cron-triggers.md](../05-app-features/cron-triggers.md)).
5. Rebuild Vercel firewall and bot rules as Cloudflare WAF rules (below).
6. Optional dress rehearsal: a `staging.` host on a preview Worker ([staging-and-preview-workers.md](../06-operations/staging-and-preview-workers.md)).
7. Lower DNS TTLs a day ahead, export the Vercel zone, build the Cloudflare zone, switch nameservers ([custom-domain-and-dns-cutover.md](../04-domains/custom-domain-and-dns-cutover.md), Vercel specifics below).
8. In the same change as the cutover, turn the Worker's crons on and remove `crons` from `vercel.json`.
9. Keep the Vercel project running, Git deploys off, for at least a few hours. Pause it later (see the stale-DNS 503 below).

## DNS and nameservers from Vercel

- Export the records with `vercel dns ls your-domain.com`. It does not list Vercel's default CAA records (`letsencrypt.org`, `pki.goog`, `sectigo.com`) as yours. Copy them into the Cloudflare zone anyway, together with any explicit CAA (`0 issue "amazon.com"`), or certificate issuance can be blocked.
- Cloudflare's dashboard quick scan imports Vercel's IPs as proxied A records and can miss Resend (mail) records. Delete what it imported and upload a zone file with "Proxy imported DNS records" unticked: [email-dns-records.md](../04-domains/email-dns-records.md).
- Keep the `_vercel` TXT records (`vc-domain-verify=your-domain.com,<ID>` and one per subdomain). They keep Vercel's claim on the domain valid, so a rollback is a DNS change, not a re-verification.

  ```
  _vercel.your-domain.com.  300 IN TXT "vc-domain-verify=your-domain.com,<ID>"
  _vercel.your-domain.com.  300 IN TXT "vc-domain-verify=www.your-domain.com,<ID>"
  ```
- Vercel sends `www` to the apex with a 308 by default. A Worker custom domain serves one hostname, so add the redirect yourself: [www-redirect-rule.md](../04-domains/www-redirect-rule.md), or the `worker.ts` redirect in [custom-domain-and-dns-cutover.md](../04-domains/custom-domain-and-dns-cutover.md).
- If Vercel is the registrar, set the nameservers in the Vercel dashboard, or through its registrar API:

  > **Unverified.** One move used `vercel api /v1/registrar/domains/<domain>/nameservers`; the request body was not recorded and is not reproduced here. Check `vercel api --help` for your CLI version, or use the dashboard.

## Env vars and deploys

- Turn off Vercel's Git deploys in `vercel.json`: `"git": { "deploymentEnabled": false }`.
- Replace `VERCEL_ENV` with `APP_ENV` and `VERCEL_GIT_COMMIT_SHA` with `GIT_SHA`, both passed at deploy (`--var=APP_ENV:production --var=GIT_SHA:...`). The client bundle gets them as `NEXT_PUBLIC_APP_ENV` and `NEXT_PUBLIC_GIT_SHA`: [deploy-from-github-actions.md](../01-start/deploy-from-github-actions.md).
- Vercel's project env vars split in two on Workers. Build-time keys (`NEXT_PUBLIC_*` and anything `next build` reads while prerendering) go to the CI env; runtime keys become Worker secrets: [secrets-and-env.md](../01-start/secrets-and-env.md). Set them from your secret manager, not by copying values out of the Vercel dashboard.
- A commit-author rewrite that only existed to get builds past Vercel's Hobby plan can go.
- An auth callback that redirects to `request.url`'s origin lands on the wrong host. On Vercel it showed as the bare `*.vercel.app` domain behind a rewrite; on Workers as `workers.dev`. Redirect to `NEXT_PUBLIC_APP_URL`: [supabase-auth-on-workers.md](../05-app-features/supabase-auth-on-workers.md).

## Crons

- Copy the schedules from `vercel.json` `crons` into `wrangler.jsonc` `triggers.crons`. The syntax is the same five-field UTC.
- Vercel Cron's `Authorization: Bearer $CRON_SECRET` convention carries over unchanged; the `scheduled()` handler sends the same header: [cron-triggers.md](../05-app-features/cron-triggers.md).
- Enable the Worker triggers only at cutover. While both are live, every job runs twice.
- After the cutover is stable, delete `crons` from `vercel.json`, or the whole file if nothing else uses it.
- A route with `maxDuration = 300` has no wall-clock cap on Workers for I/O-bound work. CPU is what is limited: set `limits.cpu_ms`.
- Change failure-alert text that points at Vercel's logs to Workers Logs.

## Firewall and bot rules

Vercel firewall rules and bot protection do not move. Rebuild them as Cloudflare WAF rules before the cutover. Let non-browser routes through (`/api/*`, `*.txt`, `*.xml`), or crawlers, feeds and webhooks get challenged. A custom rule that skips those paths works on the Free and Pro WAF.

## What gets worse

| Area | Vercel | Workers |
|---|---|---|
| Uncached render | about 1.8 s | 2.7 to 7 s; fresh-instance misses 3 to 4 s |
| Image processing | `sharp`, native | WASM only, slower: [images-without-sharp.md](../05-app-features/images-without-sharp.md) |
| Database | direct Postgres works | Hyperdrive, per-request connections, pinned placement: [hyperdrive-supabase-postgres.md](../02-data/hyperdrive-supabase-postgres.md) |
| Branch previews | a URL per branch, with deployment protection | none for a Worker with Durable Objects (OpenNext's cache has two); one shared preview Worker, unprotected unless you add `noindex` and auth: [staging-and-preview-workers.md](../06-operations/staging-and-preview-workers.md) |
| Bundle and memory | no practical limit | 64 MiB uncompressed, 128 MB per isolate, 1 s startup: [plans-and-limits.md](../01-start/plans-and-limits.md) |
| Middleware | native | `proxy.ts` labelled experimental in OpenNext |
| Customer root domains (SaaS) | an A record works | Cloudflare for SaaS needs a CNAME at `@`, so only DNS providers with CNAME flattening work: [cloudflare-for-saas-custom-hostnames.md](../04-domains/cloudflare-for-saas-custom-hostnames.md) |

Cached pages got faster (1.8 s to 0.27 to 0.4 s from India) once public pages moved into Workers Cache: [workers-cache-edge-caching.md](../03-caching/workers-cache-edge-caching.md). `"use cache"` entries, per instance on Vercel, became shared across instances. An earlier OpenNext deployment (July 2026) moved back to Vercel, citing build, runtime and media-path friction; budget for that risk.

## The stale-DNS 503

After the nameserver switch, ISP and machine DNS caches keep sending some visitors to Vercel for up to a day or two. If the Vercel project is paused, those visitors get a 503 for hours. Leave the project running, Git deploys off, for at least a few hours after the DNS change, and longer if the site has real traffic. A site with no visitors yet can pause it at once.

## Verify

```bash
dig +short NS your-domain.com                       # Cloudflare's two nameservers
dig +short CAA your-domain.com                      # the CAs you copied
curl -sI https://your-domain.com/ | grep -iE "^server|cf-ray"   # server: cloudflare
vercel dns ls your-domain.com                       # _vercel TXT still listed, for rollback
```

Then watch the error tracker and `wrangler tail` for 48 hours, and check that every cron fired once, not twice.

## Rollback

For each moved name, detach the Worker custom domain and re-add its record to Vercel from your export; for `www`, delete the Redirect Rule. With the `_vercel` TXT records kept, Vercel accepts the domain again. Switching the nameservers back is the full undo.
