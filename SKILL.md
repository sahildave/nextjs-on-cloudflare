---
name: cloudflare-nextjs-cookbook
description: Production recipes for deploying and running Next.js 16 on Cloudflare Workers with OpenNext (@opennextjs/cloudflare). Use when deploying Next.js to Workers, configuring wrangler.jsonc or worker.ts, connecting Postgres or Supabase through Hyperdrive, caching pages in Workers Cache or purging them, running Payload CMS or Supabase Auth on Workers, cron triggers, custom domains and DNS cutover, Cloudflare for SaaS customer domains, or moving a Next.js app off Vercel. Also use when a Worker returns error 1102, 500 "Worker's code had hung", "Cannot perform I/O on behalf of a different request", "Failed to parse postponed state", or sharp fails to load.
---
# Next.js on Cloudflare Workers

Pick the one recipe whose trigger matches the task and read only that file. Each recipe is self-contained, lists its prerequisites, and ends with Verify and Gotchas. Test any code marked **Unverified**, **Hunk only** or **Not run live** before shipping it. The full index, with what each recipe gets you and the order for a new app, is in [AGENTS.md](AGENTS.md).

| Use when | Recipe |
|---|---|
| First deploy of a Next.js 16 app; `wrangler.jsonc`, `worker.ts`, bundle over 64 MiB, build out of heap | [01-start/nextjs-on-workers-opennext.md](01-start/nextjs-on-workers-opennext.md) |
| Cached pages 200 but the rest 503 / error 1102; crons missing; startup limit | [01-start/plans-and-limits.md](01-start/plans-and-limits.md) |
| CI deploys from GitHub Actions: `main` to production, branches to a preview Worker | [01-start/deploy-from-github-actions.md](01-start/deploy-from-github-actions.md) |
| Build-time vs runtime env, `wrangler secret bulk` from a secret manager | [01-start/secrets-and-env.md](01-start/secrets-and-env.md) |
| Postgres/Supabase from a Worker; "different request" I/O error, `EMAXCONNSESSION`, slow region | [02-data/hyperdrive-supabase-postgres.md](02-data/hyperdrive-supabase-postgres.md) |
| postgres.js; "This socket has been closed" after `sql.end()` | [02-data/postgres-js-on-workers.md](02-data/postgres-js-on-workers.md) |
| R2 incremental-cache bucket has no entry for a page that just rendered | [02-data/r2-put-waituntil.md](02-data/r2-put-waituntil.md) |
| Serve public pages from Workers Cache (allowlist); home page never caches | [03-caching/workers-cache-edge-caching.md](03-caching/workers-cache-edge-caching.md) |
| Old page shows after a CMS publish or write | [03-caching/cache-invalidation-and-purge.md](03-caching/cache-invalidation-and-purge.md) |
| 500 "Worker's code had hung" after `revalidateTag`, rollback does not help | [03-caching/cache-components-ppr-on-workers.md](03-caching/cache-components-ppr-on-workers.md) |
| "Failed to parse postponed state"; shell replay falls back to client rendering | [03-caching/opennext-prerender-resume-fixes.md](03-caching/opennext-prerender-resume-fixes.md) |
| Page stalls at a `"use cache"` section after a client disconnect (Next patch) | [03-caching/use-cache-fill-hang-patch.md](03-caching/use-cache-fill-hang-patch.md) |
| Workers Cache on a Worker that also serves customer domains | [03-caching/workers-cache-host-gateway.md](03-caching/workers-cache-host-gateway.md) |
| Your own domain onto a Worker, nameserver switch, error 1042 | [04-domains/custom-domain-and-dns-cutover.md](04-domains/custom-domain-and-dns-cutover.md) |
| `www` to apex 308 | [04-domains/www-redirect-rule.md](04-domains/www-redirect-rule.md) |
| Mail (Resend), CAA and verification records during a zone move | [04-domains/email-dns-records.md](04-domains/email-dns-records.md) |
| Customers' own domains on your Worker (Cloudflare for SaaS) | [04-domains/cloudflare-for-saas-custom-hostnames.md](04-domains/cloudflare-for-saas-custom-hostnames.md) |
| Pick a tenant from the `Host` header | [04-domains/host-based-tenant-routing.md](04-domains/host-based-tenant-routing.md) |
| `sharp` fails; resize or re-encode images on Workers | [05-app-features/images-without-sharp.md](05-app-features/images-without-sharp.md) |
| Payload 3 on Workers; `/cms` login rejects a correct password | [05-app-features/payload-cms-on-workers.md](05-app-features/payload-cms-on-workers.md) |
| Supabase Auth with cached public pages; OAuth lands on the wrong host | [05-app-features/supabase-auth-on-workers.md](05-app-features/supabase-auth-on-workers.md) |
| Scheduled jobs on the Worker | [05-app-features/cron-triggers.md](05-app-features/cron-triggers.md) |
| Outbound API 403 only on the Worker | [05-app-features/outbound-fetch-user-agent.md](05-app-features/outbound-fetch-user-agent.md) |
| Logs, `wrangler tail`, error tracking, placement headers | [06-operations/observability-and-logs.md](06-operations/observability-and-logs.md) |
| A staging or preview Worker per branch | [06-operations/staging-and-preview-workers.md](06-operations/staging-and-preview-workers.md) |
| Production is down and the cause is unknown | [06-operations/incident-playbook.md](06-operations/incident-playbook.md) |
| The app runs on Vercel today | [migrate-from-vercel/README.md](migrate-from-vercel/README.md) |
