# Next.js on Cloudflare Workers: cookbook index

Recipes for deploying and running Next.js 16 apps on Cloudflare Workers with OpenNext, taken from three apps that run in production. Each recipe is one Markdown file with Prerequisites, Steps, Code, Verify and Gotchas.

Versions: Next 16.3.x (16.3.8 and 16.3.6), `@opennextjs/cloudflare` 1.20.15, `wrangler` 4.156.2, Node 24, pnpm 10, Payload 3, `postgres` 3.4.9. Checked 3 Nov 2026.

## Rules for agents

- Load one recipe at a time: the one whose trigger matches the task. Open another only when the current one links to it for a step you are doing.
- Each recipe is self-contained. Its Prerequisites list the recipes it assumes; you do not need to read the folder in order.
- Code marked **Unverified**, **Hunk only** or **Not run live** has not been run in the form shown. Test it before you ship it, and run the recipe's Verify section after every change.
- Versions matter. The patches target exact package versions; regenerate them against yours.
- Coming from Vercel: read [migrate-from-vercel/README.md](migrate-from-vercel/README.md) in addition to the general recipes.

## Start here: a new Next.js app on Workers

1. [01-start/plans-and-limits.md](01-start/plans-and-limits.md)
2. [01-start/nextjs-on-workers-opennext.md](01-start/nextjs-on-workers-opennext.md)
3. [03-caching/opennext-prerender-resume-fixes.md](03-caching/opennext-prerender-resume-fixes.md) and [03-caching/cache-components-ppr-on-workers.md](03-caching/cache-components-ppr-on-workers.md), if `cacheComponents` is on
4. [02-data/hyperdrive-supabase-postgres.md](02-data/hyperdrive-supabase-postgres.md), if the app uses Postgres
5. [01-start/secrets-and-env.md](01-start/secrets-and-env.md)
6. [01-start/deploy-from-github-actions.md](01-start/deploy-from-github-actions.md)
7. [06-operations/staging-and-preview-workers.md](06-operations/staging-and-preview-workers.md), if branches need a deploy target
8. [03-caching/workers-cache-edge-caching.md](03-caching/workers-cache-edge-caching.md), then [03-caching/cache-invalidation-and-purge.md](03-caching/cache-invalidation-and-purge.md)
9. [04-domains/custom-domain-and-dns-cutover.md](04-domains/custom-domain-and-dns-cutover.md)
10. [06-operations/observability-and-logs.md](06-operations/observability-and-logs.md)

## Recipes

### 01-start

- **[nextjs-on-workers-opennext](01-start/nextjs-on-workers-opennext.md)**. Use when putting a Next.js 16 app on Workers for the first time, or when `wrangler.jsonc`, `open-next.config.ts`, `worker.ts`, the bundle size (64 MiB) or a build heap error is the question. Gets you: the base config every other recipe builds on.
- **[plans-and-limits](01-start/plans-and-limits.md)**. Use when cached pages return 200 but everything else returns 503 / error 1102, cron triggers silently go missing, or a deploy fails on startup time. Gets you: what needs Workers Paid and the size, CPU, memory, startup and cron limits.
- **[deploy-from-github-actions](01-start/deploy-from-github-actions.md)**. Use when setting up CI deploys: `main` to production, branches to a separate preview Worker. Gets you: one workflow file with concurrency, heap and token guards.
- **[secrets-and-env](01-start/secrets-and-env.md)**. Use when deciding which env keys go to the build and which become Worker secrets, or pushing secrets from a secret manager with `wrangler secret bulk`. Gets you: a CI export step and a push script that prints key names only.

### 02-data

- **[hyperdrive-supabase-postgres](02-data/hyperdrive-supabase-postgres.md)**. Use when Postgres from a Worker fails with "Cannot perform I/O on behalf of a different request", "Connection terminated unexpectedly", `EMAXCONNSESSION`, or slow queries from the wrong region. Gets you: Hyperdrive config, pinned placement and a request-scoped `pg` pool.
- **[postgres-js-on-workers](02-data/postgres-js-on-workers.md)**. Use when the app uses `postgres` (postgres.js) and `wrangler tail` shows `Unhandled rejection: This socket has been closed` after `sql.end()`. Gets you: a `withSql` wrapper and the `postgres@3.4.9` patch.
- **[r2-put-waituntil](02-data/r2-put-waituntil.md)**. Use when the R2 incremental-cache bucket has no entry for a page that just rendered. Gets you: a Proxy that holds each R2 `put` open with `ctx.waitUntil`.

### 03-caching

- **[workers-cache-edge-caching](03-caching/workers-cache-edge-caching.md)**. Use when public pages should be served from Workers Cache without running the Worker, or the home page never caches because Next sends `private, no-store`. Gets you: an allowlist of shared paths (a denylist variant with its drift test), `Cache-Control` rules and the `Vary` header Next's prefetches need.
- **[cache-invalidation-and-purge](03-caching/cache-invalidation-and-purge.md)**. Use when a CMS publish or other write still shows the old page, or tag-cache lookups are slow. Gets you: `expireTagsAndPurge` (tag write, purge, second purge, home warm), the regional tag-cache default, and Durable Objects placed near the Worker.
- **[cache-components-ppr-on-workers](03-caching/cache-components-ppr-on-workers.md)**. Use when pages return 500 "Worker's code had hung" (or 524) after a `revalidateTag`, a rollback does not help, or a page gives no first byte for minutes with `cacheComponents` on. Gets you: the opennextjs-cloudflare#1318 backport (an open PR, pinned to a commit) as a pnpm patch and a local hang test with the exact SQL.
- **[opennext-prerender-resume-fixes](03-caching/opennext-prerender-resume-fixes.md)**. Use when logs show `Failed to parse postponed state`, or a prerendered shell's replay fails and the page falls back to client rendering. Gets you: `maxPostponedStateSize` and the esbuild `keepNames` patch.
- **[use-cache-fill-hang-patch](03-caching/use-cache-fill-hang-patch.md)**. Use when a page renders its shell, then stalls at one `"use cache"` section after a client disconnected, until a redeploy. Gets you: a Next patch that should stop joiners waiting on a cancelled fill (after-patch run not recorded).
- **[workers-cache-host-gateway](03-caching/workers-cache-host-gateway.md)**. Use when one Worker serves both your own host and customers' custom domains and you want Workers Cache on your host only. Gets you: an uncached gateway entrypoint in front of a cached one.

### 04-domains

- **[custom-domain-and-dns-cutover](04-domains/custom-domain-and-dns-cutover.md)**. Use when moving your own domain onto a Worker: zone setup, record export, Worker custom domains, nameserver switch, rollback, error 1042 on `workers.dev`. Gets you: the cutover steps and checks.
- **[www-redirect-rule](04-domains/www-redirect-rule.md)**. Use when `www.` should 308 to the apex without starting the Worker. Gets you: a Cloudflare Redirect Rule and its curl check.
- **[email-dns-records](04-domains/email-dns-records.md)**. Use when moving a zone that sends mail (Resend) or has verification and CAA records. Gets you: a hand-checked zone file and the records the quick scan misses.
- **[cloudflare-for-saas-custom-hostnames](04-domains/cloudflare-for-saas-custom-hostnames.md)**. Use when customers should serve your app on their own domain (app.acme.com) with certificates from Cloudflare. Gets you: the custom-hostnames API client, its tests and the zone setup.
- **[host-based-tenant-routing](04-domains/host-based-tenant-routing.md)**. Use when the app must pick a tenant from the `Host` header and expire that lookup when a domain is attached or removed. Gets you: a `proxy.ts` rewrite and a per-host cached lookup.

### 05-app-features

- **[images-without-sharp](05-app-features/images-without-sharp.md)**. Use when `sharp` fails on Workers, or server code resizes, re-encodes or rasterises images. Gets you: a WASM codec module (Photon, resvg, jSquash WebP) with EXIF rotation and a 40 MP guard.
- **[payload-cms-on-workers](05-app-features/payload-cms-on-workers.md)**. Use when running Payload 3 inside the Worker, or `/cms` login says "email or password incorrect" for a correct password. Gets you: the PBKDF2 fix, the pool wiring, R2 uploads and cache-purging hooks.
- **[supabase-auth-on-workers](05-app-features/supabase-auth-on-workers.md)**. Use when adding Supabase sign-in to a Worker whose public pages are cached, or the avatar is wrong on cached pages, or OAuth lands on the wrong host. Gets you: proxy session refresh, a page-readable hint cookie and host-safe callbacks.
- **[cron-triggers](05-app-features/cron-triggers.md)**. Use when scheduled jobs (`/api/cron/*` with a bearer check) should run on the Worker. Gets you: `triggers.crons`, a `scheduled()` handler and a failure email.
- **[outbound-fetch-user-agent](05-app-features/outbound-fetch-user-agent.md)**. Use when an outbound API (GitHub) answers 403 only on the Worker. Gets you: a `fetch` wrapper that adds a `User-Agent`.

### 06-operations

- **[observability-and-logs](06-operations/observability-and-logs.md)**. Use when turning on Workers Logs, tailing production, reading `cf-placement` and cache headers, or sending server errors to PostHog without preview deploys filing production issues. Gets you: config, commands and error-tracker wiring.
- **[staging-and-preview-workers](06-operations/staging-and-preview-workers.md)**. Use when every branch needs a deploy target but per-version preview URLs are unavailable (Durable Objects). Gets you: one `-preview` Worker with its own database, cache bucket and `staging.` host.
- **[incident-playbook](06-operations/incident-playbook.md)**. Use when production returns 5xx, hangs, serves a stale page or refuses a login and you do not yet know which recipe applies. Gets you: the first five commands and symptom to cause to fix for every incident seen.

### migrate-from-vercel

- **[README](migrate-from-vercel/README.md)**. Use when the app runs on Vercel today: cutover order, `vercel dns ls` and default CAA records, `_vercel` TXT, `VERCEL_ENV`, `vercel.json` crons, firewall rules, what gets worse, the stale-DNS 503. Gets you: the Vercel-only steps around the general recipes.
- **[case-study-nextjs-payload-supabase](migrate-from-vercel/case-study-nextjs-payload-supabase.md)**. Use when you want the full story of one move (timeline, decisions, measured Vercel vs Workers numbers), not instructions. Gets you: context and links to each recipe.
