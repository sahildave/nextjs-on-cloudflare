---
title: Cache Components and PPR on Cloudflare Workers
description: "Run Next 16 cacheComponents (PPR shells, use cache, revalidateTag) on OpenNext for Cloudflare, with the backported #1318 scheduler patch that stops a tag expiry from hanging every page."
tags: [cookbook, cloudflare]
topics: [cloudflare, cache-components, ppr, nextjs, opennext, use-cache, workerd, patch]
date: 2026-11-03
path: 03-caching/cache-components-ppr-on-workers.md
---
# Cache Components and PPR on Cloudflare Workers

You need this when a Next 16 app sets `cacheComponents: true` (partial prerendering, `"use cache"`, `cacheLife`, `revalidateTag`) and you deploy it with `@opennextjs/cloudflare`. On adapter 1.20.15 with Next 16.3.x, the first time a cache miss rebuilds a shell inside the Worker, every page that carries the expired tag hangs. This recipe is the fix: a backport of opennextjs-cloudflare#1318 as a pnpm patch, and the test that proves it before production.

This file owns the #1318 scheduler patch. The two settings that make a prerendered shell resume at all (`maxPostponedStateSize` and esbuild `keepNames`, #1465) live in [./opennext-prerender-resume-fixes.md](./opennext-prerender-resume-fixes.md). Apply that one first.

## Prerequisites

- Next 16.3.8, `@opennextjs/cloudflare` 1.20.15, wrangler 4.156.x, pnpm. Base setup: [../01-start/nextjs-on-workers-opennext.md](../01-start/nextjs-on-workers-opennext.md).
- Shells that resume: [./opennext-prerender-resume-fixes.md](./opennext-prerender-resume-fixes.md).
- The R2 / Durable Object caches that hold shells and tags: [./cache-invalidation-and-purge.md](./cache-invalidation-and-purge.md).
- Why OpenNext and not vinext: vinext's Cache Components support is incomplete (cache profiles, tags, partial shells) and it renders every page per request.

## Steps

1. Keep `cacheComponents: true` in `next.config.mjs`.
2. Build with the OpenNext CLI (`opennextjs-cloudflare build`), so shells are prerendered at build time.
3. Apply the #1318 backport to `@opennextjs/cloudflare@1.20.15` through pnpm (Code). Skip this once you run an OpenNext release that contains #1318; check the adapter's `CHANGELOG.md`.
4. Test the hang locally before you deploy (Verify).
5. Deploy, publish a change that expires a tag shared by every shell, and load pages.

## Code

### `pnpm-workspace.yaml`

```yaml
patchedDependencies:
  '@opennextjs/cloudflare@1.20.15': patches/@opennextjs__cloudflare@1.20.15.patch
```

pnpm takes one patch file per package version. A single patch file can carry both #1318 (this recipe) and the `keepNames` hunk from [./opennext-prerender-resume-fixes.md](./opennext-prerender-resume-fixes.md). If you need both, make both changes in one `pnpm patch` session.

### `patches/@opennextjs__cloudflare@1.20.15.patch`

> **Not reproduced here.** The patch is 268 lines across three files. It is a backport, so regenerate it from the upstream PR instead of copying it by hand.

```bash
pnpm patch @opennextjs/cloudflare@1.20.15      # prints a temp dir
# in that dir, apply opennextjs/opennextjs-cloudflare#1318 (scheduler + build plugin)
pnpm patch-commit '<temp dir>'
```

What it changes in the package's `dist/`:

1. `cli/templates/cache-components-scheduler.js` (new file). A drop-in `runInSequentialTasks`. Next runs each render stage in its own task and finds the end of a task with `process.nextTick`. workerd implements `nextTick` as `queueMicrotask`, so the stage ends before React has scheduled its work, the Flight row `a` is lost, and `collectSegmentData` awaits it forever. The replacement wraps `setImmediate` and counts per request; a stage has settled when an immediate runs and the request scheduled no other immediate in the meantime. See cloudflare/workerd#7687.
2. `cli/build/patches/plugins/cache-components.js` (new file). An esbuild plugin that finds Next's `runInSequentialTasks` in the app-page runtime with an ast-grep rule and replaces it with a call to the scheduler. It throws at build time if it finds no match, so a Next upgrade cannot silently skip it.
3. `cli/build/bundle-server.js`. Imports the scheduler first through an esbuild `stdin` entry (it wraps `setImmediate`, so it must load before Next captures it) and registers the plugin.

The plugin is enabled only when `cacheComponents` is on and Next is 16.2.0 or later.

## Verify

Test the hang locally. Without the patch, pages return 500 "Worker's code had hung"; with it, 200.

```bash
pnpm exec opennextjs-cloudflare build
wrangler dev --local
# In the local tag-cache state (.wrangler/state), insert a row with expire = now
# for a tag every shell carries. Then:
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8787/        # 200, not a hang
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8787/apps/x  # 200
```

The same test on a staging build: cold state plus an expired `_N_T_/...` row in the tag-cache Durable Object. Unpatched, the first request was a 500 "code had hung" and every later one hung past 45 to 60 s. Patched, every request was 200 and the next one warm.

In production: publish a change in the CMS, then check that every page stays 200 and the new content shows at once.

```bash
for p in / /categories /apps/some-app; do curl -s -o /dev/null -w "$p %{http_code} %{time_total}s\n" https://your-domain.com$p; done
```

## Gotchas

- **After `revalidateTag(tag, { expire: 0 })`, every page whose shell has the tag returns 500 "Worker's code had hung" (some run to a 524). Route handlers and the CMS still work** → with `cacheComponents`, a cache miss makes Next rebuild the shell inside the Worker, and `runInSequentialTasks` assumes Node's timer ordering. Production never hit the path until the first catalog write, because every page had a build-time shell. Apply the patch.
- **No first byte for minutes, and the invocations never show in `wrangler tail`** → the same bug, on a page with a slow `"use cache"` section. It appears whenever a shell is rebuilt (cold cache or an expired tag); workerd only sometimes cancels the request as hung.
- **A rollback does not help** → the expired tag lives in the tag-cache Durable Object, which every Worker version reads. A fresh build and deploy fixes the site (its shells are newer than the tag) until the next write.
- **1.20.15 lists Next `>=16.4.1` as supported, not 16.3.x** → it worked on 16.3.8 with the patch. Re-test on every Next or adapter bump.
- **A page that stalls at one `"use cache"` section after a client disconnects** is a different bug with its own patch: [./use-cache-fill-hang-patch.md](./use-cache-fill-hang-patch.md).
- **Marking the shared tag stale vs expired** → expiring a tag every shell carries makes the next visitor to each page wait for a full rebuild (34 s on `/`). Use `revalidateTag(tag, "max")` for it and `{ expire: 0 }` for the rest. Detail in [./cache-invalidation-and-purge.md](./cache-invalidation-and-purge.md).
- **`/` is a PPR page that refreshes inside a visitor's request** (about 8 s after a purge). Warm it. Same recipe.
- **`"use cache"` entries are shared across instances** → on Workers the cache lives in R2 plus the DO tag cache, so one instance's fill serves all. Expect that, and tag accordingly.
- **Uncached renders are slow** (2.7 to 7 s, one 34.6 s outlier). The edge cache hides it: [./workers-cache-edge-caching.md](./workers-cache-edge-caching.md).
- **Do not claim Cache Components on Workers works from a green build.** The hang appears at runtime, on the first miss. Test with an expired tag.
