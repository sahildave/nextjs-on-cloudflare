---
title: Workers Cache for a mostly-static SSR site on Cloudflare Workers
description: "Serve public Next.js pages from Workers Cache without running the Worker, with an allowlist of shared paths, the right Cache-Control, and the Vary header Next's prefetches need."
tags: [cookbook, cloudflare]
topics: [cloudflare, workers-cache, cache-control, nextjs, edge-caching, ssr]
date: 2026-11-03
path: 03-caching/workers-cache-edge-caching.md
---
# Workers Cache for a mostly-static SSR site

A content site renders the same page for most visitors. An uncached Next + Payload render on Workers cost P50 223 ms and P90 2 s of CPU, and a cold instance loads a ~50 MB bundle first. This recipe turns on Workers Cache and sets `Cache-Control` in the Worker so the pages you list as shared come from the cache and never reach it. Every other path is marked `private, no-store`. Cached pages measured 0.27 to 0.4 s from India.

## Prerequisites

- Next 16.3.8, `@opennextjs/cloudflare` 1.20.15, wrangler 4.156.x. Base setup in [`nextjs-on-workers-opennext.md`](../01-start/nextjs-on-workers-opennext.md).
- Workers Paid plan.
- Public pages that do not depend on the visitor. For example, read the signed-in user in the browser through `/api/me`, so the HTML is the same for everyone.
- To purge on writes, read [cache-invalidation-and-purge.md](./cache-invalidation-and-purge.md) next. Without it a CMS edit shows up only after `s-maxage` runs out.

## Steps

1. Enable the cache in `wrangler.jsonc`: `"cache": { "enabled": true }`.
2. Add `src/lib/worker-cache.ts` (below). It holds the shared-path allowlist and `withSharedCache(request, response)`. Keep it apart from `worker.ts` so a test can import it without the generated `.open-next/worker.js`.
3. In `worker.ts`, wrap the OpenNext handler's response: `return withSharedCache(request, await handler.fetch(request, env, ctx))`.
4. List the pages that render the same for every visitor in `SHARED_PATHS` (exact paths) and `SHARED_PREFIXES` (a path and everything under it). Every other path gets `private, no-store`, so a page you add later stays private until you list it.
5. Before you list a page, read it and every layout above it. **A page on the list that reads `cookies()`, `headers()` or the session on the server is served to every visitor**: the first visitor's render (their name, their saved items, a draft banner) is cached and handed to the next one. Next's dynamic `private, no-store` header is no signal here (Gotchas), and Workers Cache's own bypass on `Set-Cookie` and `Authorization` does not catch a page that reads a cookie and sets none. Read the viewer in the browser instead (`/api/me`).
6. Add the header tests (Code).
7. Deploy, then run the Verify commands twice, to see MISS then HIT.
8. Once the custom domain is live, set `"workers_dev": false` (see Gotchas).

## Code

### `wrangler.jsonc` (relevant part)

```jsonc
{
  "cache": { "enabled": true },
  "workers_dev": false
}
```

### `worker.ts` (relevant part)

```ts
// @ts-ignore `.open-next/worker.js` exists only after a build; worker.ts is excluded from tsconfig
import { default as handler } from "./.open-next/worker.js";
import { withSharedCache } from "./src/lib/worker-cache";

export default {
  async fetch(request, env, ctx) {
    return withSharedCache(request, await handler.fetch(request, env, ctx));
  },
} satisfies ExportedHandler;
```

### `src/lib/worker-cache.ts`

```ts
// Workers Cache headers for worker.ts, kept apart from it so a test can
// import them without the generated `.open-next/worker.js`.

/**
 * Pages that render the same for every visitor: no cookie, header or session
 * read on the server, in the page or any layout above it. Workers Cache may
 * serve these without running the Worker. Every other path is private.
 */
const SHARED_PATHS = new Set(["/", "/about", "/privacy", "/terms", "/sitemap.xml", "/robots.txt"]);
/** A prefix covers the path itself and everything under it. */
const SHARED_PREFIXES = ["/items", "/topics", "/blog", "/api/og"];

export const isShared = (path: string) =>
  SHARED_PATHS.has(path) ||
  SHARED_PREFIXES.some((prefix) => path === prefix || path.startsWith(`${prefix}/`));

/**
 * The response's Vary must name Next's router headers: a segment prefetch
 * asks for `/_tree` and each page segment at the same URL and `_rsc` hash,
 * and OpenNext's responses vary on neither, so the cache served one segment
 * for another and the client navigation stalled (a filter link froze).
 */
const ROUTER_HEADERS =
  "rsc, next-router-state-tree, next-router-prefetch, next-router-segment-prefetch, next-url";

const PRIVATE = "private, no-store";

/**
 * What Next sends on every dynamically rendered page (`searchParams`,
 * `connection()`), per-user or not, so it is no signal. A route that sets its
 * own `private` is still kept out.
 */
const NEXT_DYNAMIC = "private, no-cache, no-store, max-age=0, must-revalidate";

/** A public page with no private header: 10 minutes, a day stale. */
const SHARED = "public, max-age=0, s-maxage=600, stale-while-revalidate=86400";

/**
 * A page widened from NEXT_DYNAMIC: one minute, five stale. Next sends that
 * header on a 200 whose Suspense hole threw after the shell flushed (a
 * database hiccup), so the error boundary must not stay cached for a day.
 */
const SHARED_DYNAMIC =
  "public, max-age=0, s-maxage=60, stale-while-revalidate=300";

export function withSharedCache(
  request: Request,
  response: Response,
): Response {
  const path = new URL(request.url).pathname;
  if (!isShared(path)) {
    const marked = new Response(response.body, response);
    marked.headers.set("Cache-Control", PRIVATE);
    return marked;
  }
  // OpenNext serves the old page while one background render refreshes it
  // (a tag marked stale). Kept for s-maxage, it would hide the refreshed page.
  if (response.headers.get("x-nextjs-cache") === "STALE") {
    const stale = new Response(response.body, response);
    stale.headers.set("Cache-Control", "no-store");
    return stale;
  }
  const upstream = response.headers.get("cache-control") ?? "";
  if (
    (request.method !== "GET" && request.method !== "HEAD") ||
    response.status !== 200 ||
    response.headers.has("set-cookie") ||
    (/\b(private|no-store)\b/i.test(upstream) && upstream !== NEXT_DYNAMIC)
  )
    return response;
  const cached = new Response(response.body, response);
  const vary = cached.headers.get("vary");
  cached.headers.set(
    "vary",
    vary ? `${vary}, ${ROUTER_HEADERS}` : ROUTER_HEADERS,
  );
  cached.headers.set(
    "Cache-Control",
    upstream === NEXT_DYNAMIC ? SHARED_DYNAMIC : SHARED,
  );
  return cached;
}
```

### `scripts/worker-cache.test.ts`

```ts
import assert from "node:assert/strict";
import { test } from "node:test";
import { isShared, withSharedCache } from "../src/lib/worker-cache";

const get = (path: string) => new Request(`https://your-domain.com${path}`);
const ok = (headers: Record<string, string> = {}) =>
  new Response("body", { status: 200, headers });

test("a public page gets the shared-cache headers", () => {
  const out = withSharedCache(get("/items/x"), ok());
  assert.equal(
    out.headers.get("cache-control"),
    "public, max-age=0, s-maxage=600, stale-while-revalidate=86400",
  );
  assert.match(out.headers.get("vary") ?? "", /next-router-segment-prefetch/);
});

test("caches a public page Next rendered dynamically, briefly", () => {
  const out = withSharedCache(
    get("/"),
    ok({
      "cache-control": "private, no-cache, no-store, max-age=0, must-revalidate",
    }),
  );
  assert.equal(
    out.headers.get("cache-control"),
    "public, max-age=0, s-maxage=60, stale-while-revalidate=300",
  );
});

test("keeps a stale page out of the shared cache", () => {
  const out = withSharedCache(
    get("/topics"),
    ok({
      "x-nextjs-cache": "STALE",
      "cache-control": "s-maxage=1, stale-while-revalidate=31535999",
    }),
  );
  assert.equal(out.headers.get("cache-control"), "no-store");
});

for (const upstream of ["private, no-cache", "no-store"])
  test(`never widens an upstream "${upstream}" on a public path`, () => {
    const out = withSharedCache(get("/items/x"), ok({ "cache-control": upstream }));
    assert.equal(out.headers.get("cache-control"), upstream);
  });

for (const path of ["/dashboard", "/api/me", "/settings/profile", "/a-new-page"])
  test(`marks ${path} private, no-store`, () => {
    const out = withSharedCache(
      get(path),
      ok({ "cache-control": "public, max-age=300" }),
    );
    assert.equal(out.headers.get("cache-control"), "private, no-store");
  });

test("a prefix matches whole segments only", () => {
  assert.equal(isShared("/items"), true);
  assert.equal(isShared("/items/x"), true);
  assert.equal(isShared("/items-export"), false);
  assert.equal(isShared("/a-new-page"), false);
});
```

### Variant: a denylist, for a site where almost every page is public

If listing shared pages is impractical because nearly every page is public, invert the list: share everything except `PRIVATE_PREFIXES`. This shape fails open. A page you add and forget to list is served to every visitor, so the drift test below is mandatory, not optional.

```ts
export const PRIVATE_PREFIXES = ["/dashboard", "/settings", "/me", "/auth", "/cms", "/api/"];
const PUBLIC_API_PREFIXES = ["/api/og"];

export const isShared = (path: string) =>
  PUBLIC_API_PREFIXES.some((prefix) => path === prefix || path.startsWith(`${prefix}/`)) ||
  !PRIVATE_PREFIXES.some(
    (prefix) => path === prefix || path.startsWith(prefix.endsWith("/") ? prefix : `${prefix}/`),
  );
```

The drift test walks `src/app` for every `page.tsx`, turns the path into a URL, and fails when a page that your route table gates (a `(protected)` group, an auth route, a redirect for a logged-out visitor) still passes `isShared`. Build it against your own route table, one test per gated page:

```ts
test(`keeps gated ${url} out of the shared cache`, () =>
  assert.equal(isShared(url), false, `${url} is gated but PRIVATE_PREFIXES lets Workers Cache keep it`));
```

It catches gated pages only. A public page that personalises (reads a cookie to show a name) passes it, so the Step 5 check still applies to every page.

## Verify

```bash
curl -sI https://your-domain.com/items/some-item | grep -iE 'cache-control|cf-cache-status|vary'
# cache-control: public, max-age=0, s-maxage=600, stale-while-revalidate=86400
# cf-cache-status: MISS        (first request)
curl -sI https://your-domain.com/items/some-item | grep -i cf-cache-status
# cf-cache-status: HIT
curl -sI https://your-domain.com/dashboard | grep -i cache-control
# cache-control: private, no-store
```

Measured from India: cached pages 0.27 to 0.4 s, HIT 2 to 6 ms on repeat filter URLs, warm misses ~1 to 1.4 s median, fresh-instance misses 3 to 4 s.

## Gotchas

- **Home page never caches** → Next sends `private, no-cache, no-store, max-age=0, must-revalidate` on every dynamically rendered page, per-user or not. A rule that honours `private` drops those pages. Treat that exact string as cacheable, and honour any other `private` or `no-store` (an app that set it on purpose).
- **A broken page stuck for a day** → the same Next header appears on a 200 whose Suspense part threw after the shell flushed. Give widened pages a short life: `s-maxage=60, stale-while-revalidate=300`.
- **Filter link freezes after a client navigation** → Next sends several prefetches to one URL that differ only by header. Without the router headers in `Vary`, a cached prefetch answers the wrong one. Add them.
- **Refreshed page hidden after a write** → OpenNext answered with the stale page while a background render ran (`x-nextjs-cache: STALE`), and the Worker cached it for `s-maxage`. Send `no-store` for STALE responses.
- **Never cache** responses with `Set-Cookie`, non-GET/HEAD, or non-200.
- **Every deploy starts from an empty cache.** Workers Cache is partitioned by Worker version, so there is nothing to purge at deploy and the first minutes are all misses. A tab open across a deploy does one full reload on its next navigation (build mismatch). That is normal.
- **Each filter combination is its own URL.** The first visit is a full render (2 to 6 s from India) and later ones are a HIT. Prefetched `<Link>`s pull most combinations in before the click.
- **`*.workers.dev` served on the apex** → Workers Cache does not key on host. A `workers.dev` response (one was noindexed) can be served on the real domain, and that host sits outside the zone's WAF. Set `"workers_dev": false` once the custom domain is live. Turning custom-domain routes on turns `workers.dev` off by itself (error 1042); set `workers_dev: true` to test there before cutover, then back. A deploy clears the poisoned entries; the zone's "Purge Everything" does not.
- **A personalised page served to every visitor** → the page or a layout reads the viewer on the server and sits on the shared list (or, with the denylist variant, is missing from `PRIVATE_PREFIXES`). Take it off the list, or move the viewer read to the browser. With the denylist, run the drift test on every change.
- **App code is in the Next bundle, `worker.ts` is not.** A change under `src/` needs `opennextjs-cloudflare build`, not only `deploy`.
- **Free plan** shows cached pages as 200 and everything else as 503 / 1102. Check the plan before debugging code.
- **A `/`-specific header rule never runs** → Next sends `private, no-store` on every dynamic render, `/` included, so the widened branch is the one that applies. One app planned a separate rule for `/` and found it dead; its `/` embeds a value that stays valid for 15 minutes, which the 60 s + 300 s life fits.
- **The Worker also serves customers' domains** → the path-keyed cache can answer a customer's `/` with your landing page. Put the cache behind a host gateway: [./workers-cache-host-gateway.md](./workers-cache-host-gateway.md). The gateway runs at the placement target, so a HIT from Mumbai took about 0.29 s to first byte, against 0.14 to 0.16 s with the cache in front of the whole Worker.
