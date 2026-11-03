---
title: Workers Cache behind an uncached host gateway
description: "Share public pages through Workers Cache on your own host while a multi-tenant Worker serves customer domains that must never see those cached pages."
tags: [cookbook, cloudflare]
topics: [cloudflare, workers-cache, caching, multi-tenant, wrangler-exports, custom-hostnames]
date: 2026-11-03
path: 03-caching/workers-cache-host-gateway.md
---
# Workers Cache behind an uncached host gateway

Workers Cache (`"cache": { "enabled": true }` in `wrangler.jsonc`) is looked up before the Worker runs, and its key is the path, not the host. On a Worker that also serves customers' custom domains, a customer's `/` can then be answered with your cached landing page. This recipe puts the cache on a second entrypoint and makes the default entrypoint an uncached gateway that checks the host first. Public pages on your host hit the cache; every other host goes straight to the app.

If your Worker serves only your own host, you do not need this. Use [workers-cache-edge-caching.md](./workers-cache-edge-caching.md) instead.

## Prerequisites

- OpenNext Worker with `worker.ts` as the entry (see [nextjs-on-workers-opennext.md](../01-start/nextjs-on-workers-opennext.md)). wrangler 4.156.x, `@opennextjs/cloudflare` 1.20.15.
- Customer hostnames: [cloudflare-for-saas-custom-hostnames.md](../04-domains/cloudflare-for-saas-custom-hostnames.md).
- A list of pages that render the same for every visitor, with no cookie, session or viewer read on the server, layouts included.

## Steps

1. In `wrangler.jsonc`, enable the cache and split the entrypoints with `exports`: `default` uncached, `SharedPages` cached.
2. Add the `APP_HOST` var (your production host). `*.workers.dev` and loopback count as your own hosts without it.
3. Write the allowlist and the two helpers in `src/lib/worker-cache.ts`. Keep the file free of the `@/` alias and of imports of `.open-next/worker.js`, because wrangler bundles `worker.ts` and does not know the alias, and tests import this file without a build.
4. In `worker.ts`, export a `WorkerEntrypoint` class `SharedPages` that runs OpenNext and wraps the response in `withSharedCache`. Make the default export the gateway: redirect `www`, then forward to `SharedPages` when `forwardsToSharedCache` says yes, else run OpenNext.
5. Pass the host in `ctx.props` when calling `SharedPages`. It is part of the cache key, so `your-domain.com` and `*.workers.dev` keep separate entries.
6. Turn `workers_dev` off in production (see Gotchas).

## Code

### `wrangler.jsonc` (excerpt)

```jsonc
{
  "name": "your-worker",
  "main": "worker.ts",
  "compatibility_date": "2026-10-01",
  "compatibility_flags": ["nodejs_compat", "global_fetch_strictly_public"],
  "assets": { "directory": ".open-next/assets", "binding": "ASSETS" },
  // Workers Cache is keyed by path, not host, so it sits only in front of SharedPages.
  "cache": { "enabled": true },
  "exports": {
    "default": { "type": "worker", "cache": { "enabled": false } },
    "SharedPages": { "type": "worker", "cache": { "enabled": true } }
  },
  "routes": [
    { "pattern": "your-domain.com", "custom_domain": true },
    { "pattern": "www.your-domain.com", "custom_domain": true }
  ],
  "vars": { "APP_ENV": "preview", "APP_HOST": "your-domain.com" }
}
```

### `src/lib/worker-cache.ts`

```ts
// Relative imports only: wrangler bundles worker.ts and does not know the `@/` alias.

const SHARED_PATHS = new Set(["/", "/pricing", "/about", "/privacy", "/terms", "/sitemap.xml", "/robots.txt", "/llms.txt", "/rss.xml"]);
// Prefixes cover the path itself and everything under it.
const SHARED_PREFIXES = ["/features", "/docs", "/guides", "/compare", "/blog"];
/** The root OG image; the others sit under a shared prefix. */
const ROOT_OG_IMAGE = /^\/opengraph-image(?:-[\w-]+)?$/;
/** A status poll an embed makes. */
const EMBED_STATUS = /^\/embed\/[^/]+\/status$/;

const SHARED_PAGE = "public, max-age=0, s-maxage=600, stale-while-revalidate=86400";
const SHARED_EMBED_STATUS = "public, max-age=0, s-maxage=60";

/** The Cache-Control Workers Cache may share `path` under, or null when it must not. */
export function sharedCacheControl(path: string): string | null {
  if (EMBED_STATUS.test(path)) return SHARED_EMBED_STATUS;
  const page = path.length > 1 && path.endsWith(".md") ? path.slice(0, -3) : path; // markdown twins share with their page
  const shared = [path, page].some(
    (p) =>
      SHARED_PATHS.has(p) ||
      ROOT_OG_IMAGE.test(p) ||
      SHARED_PREFIXES.some((prefix) => p === prefix || p.startsWith(`${prefix}/`)),
  );
  return shared ? SHARED_PAGE : null;
}

/**
 * Next's router headers. A segment prefetch asks for /_tree and each page segment at the
 * same URL and _rsc hash, and OpenNext's responses vary on neither, so the cache served
 * one segment for another and client navigation stalled.
 */
const ROUTER_HEADERS = "rsc, next-router-state-tree, next-router-prefetch, next-router-segment-prefetch, next-url";

const PRIVATE = "private, no-store";

/**
 * What Next sends on every dynamically rendered page, per-user or not, so it is no signal
 * on an allowlisted path. Next sends it too on a 200 whose Suspense part crashed, so
 * such a page is shared for a minute only.
 */
const NEXT_DYNAMIC = "private, no-cache, no-store, max-age=0, must-revalidate";
const SHARED_DYNAMIC = "public, max-age=0, s-maxage=60";

/**
 * Whether the gateway sends `request` through the cached entrypoint: GET or HEAD, an
 * allowlisted path, and one of the app's own hosts. Customer hosts never qualify.
 */
export function forwardsToSharedCache(request: Request, appHost: string | undefined): boolean {
  if (request.method !== "GET" && request.method !== "HEAD") return false;
  const url = new URL(request.url);
  const host = url.hostname;
  const ownHost =
    host === "localhost" || host === "127.0.0.1" || host.endsWith(".workers.dev") || (!!appHost && host === appHost);
  return ownHost && sharedCacheControl(url.pathname) !== null;
}

/**
 * Sets the Cache-Control Workers Cache reads, on the cached entrypoint only. A response is
 * shared only when the path is allowlisted, the method is GET/HEAD, the status is 200,
 * there is no Set-Cookie, and upstream is not private/no-store (Next's dynamic default
 * aside). Everything else is marked `private, no-store`, so the cache never keeps it.
 * Nothing purges a shared page: it changes with a deploy, which starts a new cache (the
 * version is in the key).
 */
export function withSharedCache(request: Request, response: Response): Response {
  const out = new Response(response.body, response);
  const upstream = response.headers.get("cache-control") ?? "";
  const narrow = /\b(private|no-store)\b/i.test(upstream);
  const cacheControl = sharedCacheControl(new URL(request.url).pathname);
  const shared =
    (request.method === "GET" || request.method === "HEAD") &&
    response.status === 200 &&
    !response.headers.has("set-cookie") &&
    (!narrow || upstream === NEXT_DYNAMIC);

  if (cacheControl === null || !shared) {
    if (!narrow) out.headers.set("Cache-Control", PRIVATE);
    return out;
  }
  const vary = out.headers.get("vary");
  out.headers.set("vary", vary ? `${vary}, ${ROUTER_HEADERS}` : ROUTER_HEADERS);
  out.headers.set("Cache-Control", upstream === NEXT_DYNAMIC ? SHARED_DYNAMIC : cacheControl);
  return out;
}
```

### `worker.ts` (gateway part)

```ts
// @ts-ignore `.open-next/worker.js` exists only after a build; worker.ts is excluded from tsconfig
import { default as handler } from "./.open-next/worker.js";
import { WorkerEntrypoint } from "cloudflare:workers";
import { forwardsToSharedCache, withSharedCache } from "./src/lib/worker-cache";

type Env = Record<string, unknown> & { APP_HOST?: string };

/** The cached entrypoint. On a cache hit it never runs. */
export class SharedPages extends WorkerEntrypoint<Env> {
  async fetch(request: Request): Promise<Response> {
    return withSharedCache(request, await handler.fetch(request, this.env, this.ctx));
  }
}

/** The gateway, uncached: decides the host itself. */
const worker = {
  fetch(request: Request, env: Env, ctx: ExecutionContext) {
    const url = new URL(request.url);
    if (env.APP_HOST && url.hostname === `www.${env.APP_HOST}`) {
      url.hostname = env.APP_HOST;
      return Response.redirect(url.toString(), 308);
    }
    if (forwardsToSharedCache(request, env.APP_HOST)) {
      const exports = (ctx as ExecutionContext & { exports: { SharedPages: (opts: object) => Fetcher } }).exports;
      return exports.SharedPages({ props: { host: url.hostname } }).fetch(request);
    }
    return handler.fetch(request, env, ctx);
  },
};
export default worker;

// OpenNext's Durable Object classes must be exported from the entry.
// @ts-ignore generated at build time
export { DOQueueHandler, DOShardedTagCache } from "./.open-next/worker.js";
```

A production `worker.ts` usually also wraps the R2 bucket ([r2-put-waituntil.md](../02-data/r2-put-waituntil.md)) and sets a user-agent ([outbound-fetch-user-agent.md](../05-app-features/outbound-fetch-user-agent.md)).

### `src/lib/worker-cache.test.ts` (gateway routing, excerpt)

```ts
import { describe, expect, it } from "vitest";
import { forwardsToSharedCache } from "./worker-cache";

describe("forwardsToSharedCache, the gateway's routing", () => {
  const at = (url: string, method = "GET") => new Request(url, { method });

  it.each([
    "https://your-domain.com/",
    "https://your-domain.com/pricing",
    "https://your-worker.<SUBDOMAIN>.workers.dev/",
    "http://localhost:8787/robots.txt",
    "http://127.0.0.1:8787/sitemap.xml",
  ])("sends an allowlisted page on an app host to the cached entrypoint: %s", (url) => {
    expect(forwardsToSharedCache(at(url), "your-domain.com")).toBe(true);
    expect(forwardsToSharedCache(at(url, "HEAD"), "your-domain.com")).toBe(true);
  });

  it.each(["/", "/pricing", "/robots.txt", "/sitemap.xml"])(
    "sends %s on a customer's domain straight to the app",
    (path) => {
      expect(forwardsToSharedCache(at(`https://app.acme.com${path}`), "your-domain.com")).toBe(false);
      // a look-alike host that merely contains yours
      expect(forwardsToSharedCache(at(`https://your-domain.com.acme.com${path}`), "your-domain.com")).toBe(false);
    },
  );

  it("sends a POST straight to the app", () => {
    expect(forwardsToSharedCache(at("https://your-domain.com/", "POST"), "your-domain.com")).toBe(false);
  });

  it("treats no APP_HOST as no production host", () => {
    expect(forwardsToSharedCache(at("https://your-domain.com/"), undefined)).toBe(false);
  });
});
```

The full test file also covers `withSharedCache`: an allowlisted page gets the shared headers and the router headers appended to `Vary`; a year-long `s-maxage` is capped at 10 minutes; a Next-dynamic header becomes `s-maxage=60`; a private page stays `private, no-store`; a POST, a non-200 and a `Set-Cookie` response are never shared.

## Verify

```bash
# first request after a deploy is a MISS, the next is a HIT
curl -sI https://your-domain.com/pricing | grep -i cf-cache-status   # cf-cache-status: MISS
curl -sI https://your-domain.com/pricing | grep -i cf-cache-status   # cf-cache-status: HIT
# an app page that reads the viewer is never shared
curl -sI https://your-domain.com/t/some-tenant | grep -i cf-cache-status   # cf-cache-status: BYPASS
# a customer host never hits the shared cache
curl -sI https://app.acme.com/ | grep -i cf-cache-status        # absent, or not HIT
```

Measured from Mumbai against a Frankfurt-placed Worker, three curls each, first byte / total:

| Path | Before | After |
| --- | --- | --- |
| `/` | 5.14/5.29, 2.66/2.91, 0.38/0.59 s | 6.77/6.99 (MISS), 0.16/0.27, 0.16/0.28 s (HIT) |
| `/pricing` | 1.63/1.84, 0.36/0.48, 0.36/0.48 s | 3.26/3.36 (MISS), 0.22/0.33, 0.14/0.25 s (HIT) |
| `/docs/custom-domain` | 1.62/1.86, 0.41/0.65, 0.43/0.92 s | 3.88/4.29 (MISS), 0.14/0.30, 0.14/0.28 s (HIT) |

These HIT numbers were taken when the cache sat in front of the whole Worker. With the gateway in front, the gateway runs at the placement target (`cf-placement: remote-FRA`), so a HIT from Mumbai takes about 0.29 s to first byte instead of 0.14 to 0.16 s.

## Gotchas

- Cache enabled on the whole Worker → a customer's `/` could be served your cached landing page (found in review, before any customer used it). Cause: the cache key is the path and the lookup happens before any code runs. Fix: the gateway above.
- A first version marked app-host requests with a header for later code to read. Deciding the host in the gateway removed the need for it and left the proxy's headers unchanged.
- If `www` is a Worker custom domain rather than a Redirect Rule (the choice is in [nextjs-on-workers-opennext.md](../01-start/nextjs-on-workers-opennext.md)), redirect it in the gateway, as a 308, before the cache.
- Next's `private, no-cache, no-store, max-age=0, must-revalidate` also appears on a 200 whose Suspense part crashed. Sharing it for 600 s would pin the broken page. Share it for 60 s only.
- A shared page reading the viewer on the server (a cookie in a layout, a sign-in state) leaks one user's page to everyone. A landing header should read sign-in state in the browser for this reason. Check a route's layouts before adding it to the allowlist.
- `workers_dev` on in production → a `*.workers.dev` response can sit in the same cache as the real host. Turn it off for the production Worker. Keeping `ctx.props.host` in the key as well is a second guard.
- Nothing purges the shared pages. A deploy starts a cold cache, so the first request per page after a deploy is a MISS (3 to 7 s in the table above).
- Purging behind the gateway → `ctx.cache.purge()` is scoped to the entrypoint that calls it ([docs](https://developers.cloudflare.com/workers/cache/purge/)). A purge from the default entrypoint, where writes run, leaves every `SharedPages` entry in place. Fix: give `SharedPages` an RPC method that calls `this.ctx.cache.purge(...)`, and call it from the write through `ctx.exports.SharedPages({ props: { host } }).purgePaths([...])`. **Unverified**: none of the apps behind this cookbook purged a gateway cache.
- App pages that call `connection()` or read `searchParams` answer `BYPASS` and keep their normal speed; do not add them to the allowlist to "fix" that.
