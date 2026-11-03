---
title: Incident playbook for a Next.js app on Cloudflare Workers
description: "Symptom to cause to fix for the outages and near-misses hit running Next.js apps on Workers, plus the first five commands to run when production misbehaves."
tags: [cookbook, cloudflare]
topics: [cloudflare, incident, debugging, workers, runbook, nextjs, payload, hyperdrive]
date: 2026-11-03
path: 06-operations/incident-playbook.md
---
# Incident playbook for a Next.js app on Cloudflare Workers

Read this when a site on OpenNext / Workers returns 5xx, hangs, serves a stale page or refuses a login. It lists what broke, how to tell, and the fix, newest incident first. Each entry names the recipe with the full code.

## Prerequisites

- `wrangler` 4.156.x logged into the account, and the Worker's name.
- Next 16.3.8, `@opennextjs/cloudflare` 1.20.15 (the versions these notes come from).
- Observability on (`"observability": { "enabled": true }` in `wrangler.jsonc`). See [observability-and-logs.md](./observability-and-logs.md).

## Steps

First five minutes, in order:

1. Look at live logs and note the Worker version on the failing requests.
   ```bash
   wrangler tail <worker-name> --format=pretty
   ```
2. Check the plan. Free plan: cached pages return 200 and everything else 503 / error 1102.
3. Check whether it is every page or a class of pages. Route handlers and `/cms` fine but pages hung: cache-components (entry 1). Only login fails: PBKDF2 (entry 3). Only the first request after a deploy is slow: expected.
4. Check the cache layer by header.
   ```bash
   curl -sI https://your-domain.com/path | grep -iE 'cf-cache-status|cache-control|cf-placement|x-nextjs-cache'
   ```
5. Rollback only if the cause is in the code, not in the data. A tag expiry or a database change survives a rollback (entry 1).

## Code

Entries, newest first.

### 1. A CMS save hung every page (3 Nov 2026)

- **Symptom:** after `revalidateTag("content", { expire: 0 })`, every page whose shell carries the tag returned 500 "Worker's code had hung" (some ran to a 524). Route handlers and `/cms` kept working. Another app saw the same bug as no first byte for minutes, with nothing in `wrangler tail`.
- **Cause:** a cache miss rebuilds the shell inside the Worker, and Next's render scheduler assumes Node's `nextTick`. Production never hit it until the first content write.
- **A rollback does not help:** the expired tag lives in the tag-cache Durable Object, which every version reads. A fresh build and deploy restores the site until the next write.
- **Fix and pre-production test:** the #1318 backport in [../03-caching/cache-components-ppr-on-workers.md](../03-caching/cache-components-ppr-on-workers.md).
- **Related:** a page that stalls at one `"use cache"` section after a client disconnects is a different bug: [../03-caching/use-cache-fill-hang-patch.md](../03-caching/use-cache-fill-hang-patch.md).

### 2. Stale page after a write

- **Symptom:** an editor publishes, the old page keeps showing.
- **Cause, one of:** the purge ran before the tag write (order), a second data center read the old tag time from its 5 s regional cache and put the page back, or the write skipped Payload's hooks (a direct `payload.db.*` call).
- **Fix:** `expireTagsAndPurge` awaits the tag write, purges, purges again at 7 s, and warms `/`. Call it after a direct DB write. Recipe: [`cache-invalidation-and-purge.md`](../03-caching/cache-invalidation-and-purge.md).
- **Stopgap:** the dashboard's "Purge Everything", or a new deploy. A deploy clears everything because Workers Cache is partitioned by Worker version.

### 3. `/cms` login says "email or password incorrect"

- **Cause:** Payload 3 hashes at 600,000 PBKDF2 iterations. A deployed Worker caps PBKDF2 at 100,000 (observed, not documented), and Payload swallows the error.
- **Fix:** wrap `crypto.pbkdf2` in `worker.ts` and route calls above the cap to `@noble/hashes` (same bytes as Node). Local workerd and miniflare have no cap, so only `wrangler dev --remote` shows it. A login then costs ~15 s of CPU.
- **Follow-on:** workers freeze the clock during synchronous CPU. After that hash, pg's 10 s connect timeout fired instantly ("cannot begin transaction: Connection terminated unexpectedly"). Await one `setTimeout(0)` after long synchronous work. Recipe: [`payload-cms-on-workers.md`](../05-app-features/payload-cms-on-workers.md).

### 4. Everything except cached pages returns 503 / 1102

- **Cause:** Workers Free caps CPU at 10 ms per request.
- **Fix:** Workers Paid. See [`plans-and-limits.md`](../01-start/plans-and-limits.md).

### 5. Database errors

| Symptom | Cause | Fix |
|---|---|---|
| "Cannot perform I/O on behalf of a different request", or a hang under burst | A pg pool shared across requests | Request-scoped pool |
| `EMAXCONNSESSION` | Hyperdrive's origin limit (60 observed without one, not documented; `wrangler hyperdrive get`) above Supabase's session pooler (15) | `wrangler hyperdrive update <ID> --origin-connection-limit=10` |
| "Connection terminated unexpectedly" on every query | Direct TLS to Supabase's pooler | Use Hyperdrive |
| `unhandledRejection: This socket has been closed`, ~5k/hour | Per-query `pg.Client` with no `'error'` listener | Add the listener before `connect()` |
| Same, from `postgres.js` | Unguarded `tcp.emit('error')` in `cf/polyfills.js` | Patch `postgres@3.4.9` ([../02-data/postgres-js-on-workers.md](../02-data/postgres-js-on-workers.md)) |
| Queries slow, every page 2 to 3 s | Worker placed in Mumbai, database in Frankfurt | `placement.region: "aws:eu-central-1"`; check `cf-placement` |
| Deploy or preview will not start | No `localConnectionString` on the Hyperdrive binding | Add it |

Recipe: [`hyperdrive-supabase-postgres.md`](../02-data/hyperdrive-supabase-postgres.md).

### 6. Custom domain problems

- **Error 1042 everywhere after adding custom-domain routes** → that turned `workers.dev` off. Set `"workers_dev": true` to test there before cutover, then back to `false`.
- **A `workers.dev` response (noindexed) served on the apex** → Workers Cache does not key on host. Deploy to clear it; "Purge Everything" does not.
- **No address from resolvers that follow Cloudflare's nameservers** → a pending zone serves no Worker custom-domain records. It took ~40 min to go active and ~2 min more for the certificate here. Press "Check nameservers now" right after the switch.
- **Old addresses linger** → ISP and machine DNS caches keep the previous host for up to a day or two. Don't pause the old project at cutover if anyone uses the site.
- **Certificates fail to issue** → the old host's default CAA records were not copied. Export the zone as a BIND file before the move and copy `letsencrypt.org`, `pki.goog` and `sectigo.com`.
- **Stale-DNS visitors get a 503 for hours** → the old project was paused at once. Leave it running a few hours after the DNS change.
- **Email stopped** → the dashboard's DNS quick scan imports proxied A records and can miss Resend / email records. Check the list or upload a zone file.

### 7. Outbound API calls fail only on the Worker

- **Every GitHub-backed page looks private, or an API answers 403** → workerd's `fetch` sends no `User-Agent`. Fix: [../05-app-features/outbound-fetch-user-agent.md](../05-app-features/outbound-fetch-user-agent.md).

### 8. Deploy and build

- **`next build` out of heap on `ubuntu-latest`** → Node's ~2 GB default. `NODE_OPTIONS: --max-old-space-size=6144` on the build step.
- **Build prerender timed out against the live database** → retry before debugging.
- **Source-map upload fails** → over the 60 MiB limit. `"upload_source_maps": false`.
- **Free plan skips cron triggers** → 5 per account; deploy silently skips the rest.
- **A cron ran twice** → crons were enabled on the Worker before the cutover while the old host's crons still ran. Enable them at cutover.
- **"Unexpected deploy" in the dashboard** → a secret change creates a new Worker version.
- **A fix is not live** → app code compiles into the Next bundle, `worker.ts` does not. A change under `src/` needs `opennextjs-cloudflare build`, not only `deploy`. Confirm the new version in `wrangler tail`.
- **Wrangler's "add it on your behalf?" prompt** rewrites and reformats `wrangler.jsonc`. Re-read the file before editing.
- **Shared-database migration** the Worker depends on arrived with a merge → run migrations before the code deploys.

### 9. Images

- **`sharp` missing** → port to WASM. Photon ignores EXIF rotation (the recipe rotates from the header itself) and resvg loads no fonts, so `<text>` in an SVG icon disappears. Recipe: [`images-without-sharp.md`](../05-app-features/images-without-sharp.md).
- **OpenNext mangles bare `.wasm` imports** and webpack refuses `.wasm` in server code. Use `webpackIgnore` and `serverExternalPackages`; a package with a relative `./x.wasm` import in its workerd entry works.

## Verify

After any fix, confirm on the live site, not the build:

```bash
curl -s -o /dev/null -w '%{http_code}\n' https://your-domain.com/
curl -sI https://your-domain.com/ | grep -iE 'cf-cache-status|cf-placement'
wrangler tail <worker-name> --format=pretty    # no hung, 1102, unhandledRejection or purge-skipped lines
```

## Gotchas

- **Don't tune a render to fix a cache problem.** Uncached renders cost P50 223 ms and P90 2 s of CPU, and fresh instances 3 to 4 s. Skip the render instead ([`workers-cache-edge-caching.md`](../03-caching/workers-cache-edge-caching.md)).
- **The first minutes after a deploy are all misses.** Not an incident.
- **A tab open across a deploy** does one full reload on its next navigation. Normal.
- **A test that passes in local workerd can still fail in production.** The PBKDF2 cap and the hang both needed `wrangler dev --remote` or an expired-tag test.
- **Bot protection doesn't move.** Rebuild the old host's firewall rules as Cloudflare WAF rules and keep non-browser routes (`/api/*`, `.txt`, `.xml`) let through.
- **Do not build anything that only works on Enterprise** if that tier is temporary (a trial or a credit program). It stops when the tier does.
- **Persistent Postgres sockets from a serverless runtime to Supabase's pooler can hang for 300 s.** Supabase's HTTP Data API avoids that, but Payload cannot use it, so Hyperdrive holds the pool; keep non-Payload tables on `supabase-js`.
