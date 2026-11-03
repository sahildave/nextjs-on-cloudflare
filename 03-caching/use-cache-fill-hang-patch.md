---
title: Fix 'use cache' fills that hang forever on Workers
description: "Patch Next so a request that leads a 'use cache' fill and then disconnects cannot leave every later request in that isolate waiting."
tags: [cookbook, cloudflare]
topics: [cloudflare, use-cache, cache-components, pnpm-patch, workerd, hangs]
date: 2026-11-03
path: 03-caching/use-cache-fill-hang-patch.md
---
# Fix 'use cache' fills that hang forever on Workers

Symptom: some pages render the shell, then stall at a `"use cache"` section with no error, and only a redeploy or a new isolate clears it. Cause: a request that leads a `"use cache"` fill registers it in a module-level map that later requests join. If the leader's client disconnects, workerd cancels the leader's I/O, the fill never settles, and every joiner in that isolate waits on it. This recipe is a pnpm patch to Next that turns off the cross-request dedup on workerd. Per-request dedup stays.

## Prerequisites

- Next 16 with Cache Components and `"use cache"` on OpenNext. The patch was made against `next@16.3.6` and has not been tested on 16.3.8; regenerate the patch against your exact version (see Steps).
- pnpm with `patchedDependencies`. See [cache-components-ppr-on-workers.md](./cache-components-ppr-on-workers.md) for the other patches such an app needs (the cache-components scheduler, `keepNames`).

## Steps

1. Reproduce first. In production, pages stalled at one `"use cache"` list in 4 of 7 abort rounds (a client disconnects mid-load, then the page is loaded again). Use the same loop on a page with a cold `"use cache"` section.
2. Run `pnpm patch next@<version>`, edit the two files below in the directory pnpm prints, then `pnpm patch-commit <dir>`.
3. Commit `patches/next@<version>.patch` and the `patchedDependencies` entry.
4. Bump the patch whenever Next's version changes; the line numbers move and the hunk may not apply.
5. Remove the patch when upstream fixes this. Check Next's `use-cache-wrapper` for a Workers-aware dedup first.

## Code

### `pnpm-workspace.yaml` (excerpt)

```yaml
patchedDependencies:
  next@16.3.6: patches/next@16.3.6.patch
```

### `patches/next@16.3.6.patch`

> **Proven on `next@16.3.6` only, and shown as a hunk.** It ran in production on 16.3.6; it has not been run on 16.3.8 or later. The diff below has no `diff --git` or `index` line and the CJS copy is described, not shown, so regenerate the patch with `pnpm patch` against your exact Next version.

The same hunk in two files, `dist/esm/server/use-cache/use-cache-wrapper.js` and `dist/server/use-cache/use-cache-wrapper.js`. Shown here for the ESM file; the context lines differ by a few lines in the CJS one.

```diff
--- a/dist/esm/server/use-cache/use-cache-wrapper.js
+++ b/dist/esm/server/use-cache/use-cache-wrapper.js
@@ -1991,7 +1991,10 @@ export async function cache(kind, id, boundArgsLength, originalFn, args) {
         // headers, so concurrent requests with identical request data should share
         // a fill too; that request-scoped `cacheHandlerKey` keeps requests with
         // different cookies or headers in separate entries.
-        const skipCrossRequestDedupe = isPrivate && !process.env.__NEXT_DEV_SERVER;
+        // workerd cancels a request's I/O when its client disconnects, so a leader's fill
+        // can never settle and every later joiner in that isolate waits on it forever.
+        const skipCrossRequestDedupe = isPrivate && !process.env.__NEXT_DEV_SERVER
+            || globalThis.navigator?.userAgent === "Cloudflare-Workers";
         try {
```

The file is under `node_modules/next/dist/`. Find the line with `grep -n skipCrossRequestDedupe node_modules/next/dist/**/use-cache-wrapper.js`.

## Verify

1. `pnpm install` applies the patch; `grep -n "Cloudflare-Workers" node_modules/next/dist/server/use-cache/use-cache-wrapper.js` finds the line.
2. Build and deploy, then run the abort loop from Steps against a page with a cold `"use cache"` section.
3. Expected: each following load completes. Before the patch, 4 of 7 rounds left it stalled. The 4-of-7 figure is recorded; the after-patch run is not.

## Gotchas

- The condition `navigator.userAgent === "Cloudflare-Workers"` is how workerd identifies itself to Node-compatible code. Under `next dev` and `next start` it is false, so local behaviour is unchanged.
- The dedup is a cost optimisation. With it off, N concurrent cold requests for one key run N fills inside an isolate. Cache hits are unaffected.
- A hang with no first byte at all is a different bug: the cache-components scheduler on workerd. See [cache-components-ppr-on-workers.md](./cache-components-ppr-on-workers.md).
- A patch against `dist/` breaks silently if the file moves. After a Next bump, run the grep in Verify before shipping.
- This showed up only in production abort rounds. Add an abort loop to your pre-launch checks.
