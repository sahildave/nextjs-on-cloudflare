---
title: Next.js 16 on Cloudflare Workers with OpenNext
description: "Deploy a Next.js 16 app with Cache Components to Cloudflare Workers through @opennextjs/cloudflare, with the wrangler config, worker entry, Next config and bundle-size checks every other recipe builds on."
tags: [cookbook, cloudflare]
topics: [cloudflare, workers, opennext, nextjs, cache-components, wrangler]
date: 2026-11-03
path: 01-start/nextjs-on-workers-opennext.md
---
# Next.js 16 on Cloudflare Workers with OpenNext

You have a Next.js 16 app and want it on Cloudflare Workers. This recipe is the base every other recipe sits on: the adapter, `wrangler.jsonc`, a custom `worker.ts` entry, and the Next config it needs. It was proven on a production Next 16.3.8 + Payload 3 + Supabase app on 3 Nov 2026. Moving from Vercel: also read [../migrate-from-vercel/README.md](../migrate-from-vercel/README.md).

## Prerequisites

- Workers Paid plan. The free plan caps CPU at 10 ms per request, and a Next render fails with error 1102. See [./plans-and-limits.md](./plans-and-limits.md).
- Versions used: Next 16.3.8, `@opennextjs/cloudflare` 1.20.15, `wrangler` 4.156.2, Node 24, pnpm 10.
- A database reachable from Workers (Postgres needs Hyperdrive): [../02-data/hyperdrive-supabase-postgres.md](../02-data/hyperdrive-supabase-postgres.md).
- Caching headers and purge, which make the site fast: [../03-caching/workers-cache-edge-caching.md](../03-caching/workers-cache-edge-caching.md).
- Deploying from CI: [./deploy-from-github-actions.md](./deploy-from-github-actions.md).

OpenNext over vinext: Cloudflare recommends vinext for new apps, but vinext's Cache Components support is incomplete and it renders every page per request. OpenNext adapts the real `next build` output, so `cacheComponents`, `"use cache"`, and `proxy.ts` (since adapter 1.20.x) behave as Next documents them. The opennext.js.org docs lag the adapter; read the adapter's `CHANGELOG.md`.

## Steps

1. Install the adapter and wrangler.
   ```bash
   pnpm add @opennextjs/cloudflare@1.20.15
   pnpm add -D wrangler@4.156.2
   ```
2. Add scripts to `package.json`. Keep your normal `build` (for example `next build --webpack`); the adapter runs it.
   ```json
   "cf:build": "opennextjs-cloudflare build",
   "cf:preview": "opennextjs-cloudflare preview",
   "cf:deploy": "opennextjs-cloudflare deploy"
   ```
3. Create `wrangler.jsonc` and `open-next.config.ts` (Code below). The R2 bucket for the incremental cache must exist: `pnpm exec wrangler r2 bucket create <your-app>-next-cache`.
4. Create `worker.ts` (Code below). It wraps OpenNext's generated handler so you can add headers, redirects and cron. Exclude it from `tsconfig.json`, because it imports a file that exists only after a build.
5. Edit `next.config.mjs`: set `images.unoptimized: true` and list workerd-only packages in `serverExternalPackages` (Code below).
6. Add `.open-next`, `.wrangler` and `.dev.vars*` to `.gitignore`.
7. If you use `cacheComponents`, apply the two Cache Components recipes (next section).
8. Build, then preview in the real Workers runtime.
   ```bash
   NODE_OPTIONS=--max-old-space-size=6144 pnpm exec opennextjs-cloudflare build
   pnpm exec opennextjs-cloudflare preview
   ```
9. Check the bundle size before the first deploy: `pnpm exec wrangler deploy --dry-run --outdir bundled/`. "Total Upload" must stay under 64 MiB uncompressed.

### Cache Components apps

With `cacheComponents: true`, two separate fixes are needed on adapter 1.20.15, each with its own recipe:

- Prerendered shells fall back to a full render on every request (`maxPostponedStateSize`, esbuild `keepNames`): [../03-caching/opennext-prerender-resume-fixes.md](../03-caching/opennext-prerender-resume-fixes.md).
- Every page hangs after the first tag expiry, because Next's render scheduler assumes Node's `nextTick` (the #1318 backport): [../03-caching/cache-components-ppr-on-workers.md](../03-caching/cache-components-ppr-on-workers.md).

Both changes go into the one `patches/@opennextjs__cloudflare@1.20.15.patch` file, since pnpm takes one patch per package version.

## Code

### `wrangler.jsonc`

Product values are placeholders. Bindings for Hyperdrive, cron and domains are explained in their own recipes.

```jsonc
{
	"$schema": "node_modules/wrangler/config-schema.json",
	"name": "your-app",
	"main": "worker.ts",
	"compatibility_date": "2026-10-01",
	"compatibility_flags": ["nodejs_compat", "global_fetch_strictly_public"],
	"assets": { "directory": ".open-next/assets", "binding": "ASSETS" },
	// Workers Cache: see workers-cache-edge-caching.md
	"cache": { "enabled": true },
	"observability": { "enabled": true },
	// Source maps were over the API's 60 MiB cap.
	"upload_source_maps": false,
	// Only the main-branch deploy passes --var APP_ENV:production.
	"vars": { "APP_ENV": "preview" },
	// Run next to the database, not next to the visitor: a page makes dozens of queries.
	"placement": { "region": "aws:eu-central-1" },
	"limits": { "cpu_ms": 300000 },
	"routes": [
		{ "pattern": "your-domain.com", "custom_domain": true },
		{ "pattern": "www.your-domain.com", "custom_domain": true }
	],
	// Off once the domain is live: *.workers.dev would serve the site outside the zone's WAF.
	"workers_dev": false,
	// On, so branch deploys (versions upload) get a preview URL.
	"preview_urls": true,
	"r2_buckets": [
		{ "binding": "NEXT_INC_CACHE_R2_BUCKET", "bucket_name": "your-app-next-cache" }
	],
	"durable_objects": {
		"bindings": [
			{ "name": "NEXT_CACHE_DO_QUEUE", "class_name": "DOQueueHandler" },
			{ "name": "NEXT_TAG_CACHE_DO_SHARDED", "class_name": "DOShardedTagCache" }
		]
	},
	"migrations": [
		{ "tag": "v1", "new_sqlite_classes": ["DOQueueHandler", "DOShardedTagCache"] }
	],
	"services": [
		{ "binding": "WORKER_SELF_REFERENCE", "service": "your-app" }
	]
}
```

Add the `hyperdrive` and `triggers` blocks from the recipes that cover them.

### `open-next.config.ts`

The minimal version. A busier app adds tag-cache tuning (regional cache and a Frankfurt location hint for the Durable Objects); that lives in [../03-caching/cache-invalidation-and-purge.md](../03-caching/cache-invalidation-and-purge.md).

```ts
import { defineCloudflareConfig } from "@opennextjs/cloudflare";
import r2IncrementalCache from "@opennextjs/cloudflare/overrides/incremental-cache/r2-incremental-cache";
import { withRegionalCache } from "@opennextjs/cloudflare/overrides/incremental-cache/regional-cache";
import doQueue from "@opennextjs/cloudflare/overrides/queue/do-queue";
import doShardedTagCache from "@opennextjs/cloudflare/overrides/tag-cache/do-sharded-tag-cache";

// "use cache", ISR and revalidateTag share one cache across every isolate:
// pages and data in R2 (fronted by the colo cache), tags in sharded Durable
// Objects, time-based revalidation deduplicated by the DO queue.
export default defineCloudflareConfig({
  incrementalCache: withRegionalCache(r2IncrementalCache, { mode: "long-lived" }),
  tagCache: doShardedTagCache({ baseShardSize: 12 }),
  queue: doQueue,
});
```

### `worker.ts`

```ts
// @ts-expect-error `.open-next/worker.js` is generated at build time
import { default as handler } from "./.open-next/worker.js";

export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);
    // Example: send www to the apex.
    if (url.hostname === "www.your-domain.com") {
      url.hostname = "your-domain.com";
      return Response.redirect(url.toString(), 308);
    }
    return handler.fetch(request, env, ctx);
  },
} satisfies ExportedHandler<{ WORKER_SELF_REFERENCE: Fetcher }>;

// The Durable Object classes must be exported from the entry file.
// @ts-expect-error `.open-next/worker.js` is generated at build time
export { DOQueueHandler, DOShardedTagCache } from "./.open-next/worker.js";
```

A fuller `worker.ts` also wraps the response in a cache-header function ([../03-caching/workers-cache-edge-caching.md](../03-caching/workers-cache-edge-caching.md)), installs a PBKDF2 fallback for Payload ([../05-app-features/payload-cms-on-workers.md](../05-app-features/payload-cms-on-workers.md)) and adds a `scheduled()` handler ([../05-app-features/cron-triggers.md](../05-app-features/cron-triggers.md)).

### `tsconfig.json` (excerpt)

```json
"exclude": ["node_modules", "worker.ts"]
```

### `next.config.mjs` (excerpt)

```js
const nextConfig = {
  cacheComponents: true,

  // Why: ./opennext-prerender-resume-fixes.md
  experimental: { maxPostponedStateSize: "25mb" },

  // Packages that ship workerd builds only resolve unbundled.
  serverExternalPackages: ["postgres", "@cf-wasm/photon", "@cf-wasm/resvg"],

  images: { unoptimized: true }, // no sharp on Workers
};
```

Trim `serverExternalPackages` to what you use. The WASM entries belong to [../05-app-features/images-without-sharp.md](../05-app-features/images-without-sharp.md).

### `pnpm-workspace.yaml` (excerpt)

```yaml
allowBuilds:
  esbuild: true
  workerd: true
# Only for Cache Components apps; see the two recipes linked above.
patchedDependencies:
  '@opennextjs/cloudflare@1.20.15': patches/@opennextjs__cloudflare@1.20.15.patch
```

### Runtime files that do not exist on Workers

Workers have no filesystem. Bundle files at build time instead of reading them at request time. One app read every markdown file in `next.config.mjs` and passes the text through `env`:

```js
env: { SITE_CONTENT_FILES: JSON.stringify(contentFiles()) }
```

Files the OG image reads are inlined as modules.

Watch the size: Next copies every `next.config` `env` entry into every bundle. One app inlined 1.7 MB of HTML through `env` and it cost about 7 MB of Worker. Import large text as a module instead (`import page from "./page.html" with { type: "text" }`).

## Verify

```bash
pnpm exec opennextjs-cloudflare build        # ends with ".open-next/worker.js" written
pnpm exec wrangler deploy --dry-run --outdir bundled/   # Total Upload under 64 MiB
pnpm exec opennextjs-cloudflare preview      # then:
curl -sI http://localhost:8787/ | head -1    # HTTP/1.1 200 OK
```

After deploy, `curl -sI https://your-app.<subdomain>.workers.dev/` returns 200 and a `cf-placement` header (`remote-FRA` for a Frankfurt pin).

## Gotchas

- Cached pages return 200 but everything else 503 or 1102 → you are on the free plan → upgrade to Workers Paid.
- `next build` dies with an out-of-memory error on `ubuntu-latest` → Node's default heap is about 2 GB → `NODE_OPTIONS=--max-old-space-size=6144` on the build step.
- Source map upload fails → the maps exceed the API's 60 MiB cap → `"upload_source_maps": false`, and upload maps to your error tracker instead.
- A change under `src/` does not go live after `wrangler deploy` → app code compiles into the Next bundle, `worker.ts` does not → run `opennextjs-cloudflare build` first, then check the new version in `wrangler tail`.
- Wrangler asks "add it on your behalf?" and rewrites `wrangler.jsonc` → it reformats the file → re-read the file before you edit it.
- The deploy step needs a Hyperdrive `localConnectionString` → OpenNext's cache-population step runs the Worker locally → give it a local connection string in `wrangler.jsonc`.
- Bare `.wasm` imports break → OpenNext rewrites them to a path next to the importer → use packages whose workerd entry imports `./x.wasm` relatively, or a small local wrapper package. Webpack also refuses `.wasm` in server code: use `/* webpackIgnore: true */` and `serverExternalPackages`.
- Adding custom-domain routes turns `workers.dev` off (error 1042) → routes and `workers_dev` interact → set `"workers_dev": true` to test there before cutover, then false.
- Adapter 1.20.15 lists Next `>=16.4.1` as supported, not 16.3.x → 16.3.8 and 16.3.6 both ran on it in production, with the patches above. Upgrade Next when you can.
- Bundle over 64 MiB → something pulls test or dev code into the server bundles → one app's was 70.1 MiB until `src/instrumentation.ts` stopped lazy-importing its e2e harness (a `NODE_ENV` guard), then 44.1 MiB. Check the `--dry-run` size after any change to instrumentation or `next.config` `env`.
- An outbound API answers 403 only on the Worker → workerd's `fetch` sends no `User-Agent` → [../05-app-features/outbound-fetch-user-agent.md](../05-app-features/outbound-fetch-user-agent.md).
- Host-based logic in `proxy.ts` treats the test URL as a stranger → `*.workers.dev` is not in its list of app hosts → count it as an app host.
- The preview environment's `WORKER_SELF_REFERENCE` still points at the production Worker → `services` is not inherited per environment → set it to the preview Worker's name (see [../06-operations/staging-and-preview-workers.md](../06-operations/staging-and-preview-workers.md)).
- Worker limits to design around: 64 MiB uncompressed size, 128 MB per isolate, 1 s startup. See [./plans-and-limits.md](./plans-and-limits.md).
- Known costs: uncached renders measured 2.7 to 7 s, image processing is WASM-only, and `proxy.ts` is still labelled experimental in OpenNext. The Workers cache hides most of it for visitors.
