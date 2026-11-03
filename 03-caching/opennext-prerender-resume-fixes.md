---
title: Making prerendered shells resume on Workers (OpenNext)
description: "Two settings that stop Next.js Cache Components shells from falling back to a full render on workerd: a smaller maxPostponedStateSize and esbuild keepNames."
tags: [cookbook, cloudflare]
topics: [cloudflare, opennext, cache-components, ppr, esbuild, zlib]
date: 2026-11-03
path: 03-caching/opennext-prerender-resume-fixes.md
---
# Making prerendered shells resume on Workers (OpenNext)

With Cache Components, `next build` prerenders a static shell and the server resumes it per request. On Workers two things break the resume silently: pages still render, but from scratch, and the cache stops helping. This recipe fixes both and says how to tell.

This file owns `experimental.maxPostponedStateSize` and the `keepNames` patch (opennextjs-cloudflare#1465). The separate hang when a shell is rebuilt inside the Worker (opennextjs-cloudflare#1318) is in [./cache-components-ppr-on-workers.md](./cache-components-ppr-on-workers.md).

## Prerequisites

- A Next 16 app on `@opennextjs/cloudflare` 1.20.15 with Cache Components or PPR on: [../01-start/nextjs-on-workers-opennext.md](../01-start/nextjs-on-workers-opennext.md).
- pnpm for `patchedDependencies`.

## Steps

1. Set `experimental.maxPostponedStateSize: "25mb"` in `next.config.ts` (or `.mjs`).
2. Create `patches/@opennextjs__cloudflare@1.20.15.patch` with the `keepNames` change (see Code). Generate it with `pnpm patch @opennextjs/cloudflare@1.20.15`, edit `dist/cli/build/bundle-server.js`, then `pnpm patch-commit`. pnpm takes one patch file per package version: if you also need the #1318 backport, make both changes in the same session.
3. Register the patch in `pnpm-workspace.yaml`.
4. Rebuild and redeploy. Check `wrangler tail` for the two symptoms below.
5. Drop the patch when opennextjs-cloudflare#1465 ships in your version.

## Code

### `next.config.ts` (excerpt)
```ts
experimental: {
  // Next inflates the shell's resume data with a cap of 5x this (500 MiB by
  // default); workerd's zlib rejects any cap over 128 MiB, so every resume of
  // a prerendered shell failed and fell back to a full render.
  maxPostponedStateSize: "25mb",
},
```

### `patches/@opennextjs__cloudflare@1.20.15.patch` (the `keepNames` hunk)

> **Hunk only.** Shown without the `diff --git` and `index` lines, and the line numbers differ when the #1318 change shares the file (one combined copy is at `@@ -84,6 +94,10 @@`). Regenerate it with `pnpm patch` rather than pasting it.

```diff
--- a/dist/cli/build/bundle-server.js
+++ b/dist/cli/build/bundle-server.js
@@ -84,6 +84,10 @@ export async function bundleServer(buildOpts, projectOpts) {
         minifyWhitespace: projectOpts.minify && !debug,
         minifyIdentifiers: false,
         minifySyntax: projectOpts.minify && !debug,
+        // opennextjs-cloudflare#1465: esbuild renames colliding top-level functions
+        // (`io` -> `io4`), so React's resume of a `next build` prerender no longer
+        // matches component names and falls back to client rendering.
+        keepNames: true,
         legalComments: "none",
         metafile: true,
```

### `pnpm-workspace.yaml` (excerpt)
```yaml
patchedDependencies:
  '@opennextjs/cloudflare@1.20.15': patches/@opennextjs__cloudflare@1.20.15.patch
```

## Verify

1. `wrangler tail <worker>` and request a prerendered page.
2. Expected: no `Failed to parse postponed state` line.
3. A check on a test Worker: warm public page 0.25 s with `cf-cache-status: HIT`, cold 0.6 to 1.0 s.

## Gotchas

- Symptom: every prerendered page logs `Failed to parse postponed state`, and pages render fully each time. Cause: Next's inflate cap is 5x `maxPostponedStateSize`, 500 MiB by default, and workerd's zlib rejects any cap over 128 MiB. Fix: 25mb gives a 5 x 25 = 125 cap, under 128.
- Symptom: no parse error, but React's replay of the shell fails and the page falls back to client rendering. Cause: esbuild renames colliding top-level functions (`io` becomes `io4`), including Next's `AppRouter`, so the names no longer match the prerender. Fix: `keepNames: true` in OpenNext's server bundle.
- Both showed up only as runtime errors on a deployed Worker. `next build` and `opennextjs-cloudflare build` pass either way.
- Any value of 25 or less keeps the cap (5x) under 128 MiB. Three production apps use 25mb.
