---
title: Holding OpenNext R2 cache writes open with waitUntil
description: "Wrap the R2 incremental-cache binding in a Proxy so each put survives the response closing, instead of being cancelled by workerd."
tags: [cookbook, cloudflare]
topics: [cloudflare, opennext, r2, incremental-cache, waituntil]
date: 2026-11-03
path: 02-data/r2-put-waituntil.md
---
# Holding OpenNext R2 cache writes open with waitUntil

OpenNext's R2 incremental cache does not await some of its `put` calls. When the response finishes, workerd cancels whatever is still pending, so a page you just rendered never lands in the cache and renders again on the next request. This recipe wraps the R2 binding so every `put` is held with `ctx.waitUntil`.

## Prerequisites

- OpenNext with the R2 incremental cache: [../01-start/nextjs-on-workers-opennext.md](../01-start/nextjs-on-workers-opennext.md).
- A custom `worker.ts` that wraps OpenNext's handler (for example one that adds [shared cache headers](../03-caching/workers-cache-edge-caching.md)).
- `@opennextjs/cloudflare` 1.20.15.

## Steps

1. Keep `worker.ts` as the Worker entry (`"main": "worker.ts"`).
2. Replace `env.NEXT_INC_CACHE_R2_BUCKET` with a Proxy whose `put` registers its promise with `ctx.waitUntil`.
3. Wrap `env` in a second Proxy that returns the held bucket for that key.
4. Skip the proxy when the binding is absent: `new Proxy(undefined, ...)` throws.
5. Re-export the Durable Object classes.

## Code

### `worker.ts`
```ts
// @ts-ignore `.open-next/worker.js` exists only after a build; worker.ts is excluded from tsconfig
import { default as handler } from "./.open-next/worker.js"

const worker = {
  async fetch(request: Request, env: Record<string, unknown>, ctx: ExecutionContext) {
    // OpenNext does not await some incremental-cache puts, and workerd cancels
    // them as the response closes. Hold each R2 put in `ctx.waitUntil` so it
    // finishes; the regional Cache API write is not held and can still be cancelled.
    const bucket = env.NEXT_INC_CACHE_R2_BUCKET as R2Bucket | undefined
    if (!bucket) return handler.fetch(request, env, ctx)
    const heldBucket = new Proxy(bucket, {
      get(target, prop) {
        if (prop === "put") {
          return (...args: Parameters<R2Bucket["put"]>) => {
            const put = target.put(...args)
            // The caller still sees a failure; waitUntil only keeps the Worker alive.
            ctx.waitUntil(put.catch(() => {}))
            return put
          }
        }
        // Native binding methods need the real bucket as `this`, not the Proxy.
        const value = Reflect.get(target, prop)
        return typeof value === "function" ? value.bind(target) : value
      },
    })
    const heldEnv = new Proxy(env, {
      get: (target, prop) =>
        prop === "NEXT_INC_CACHE_R2_BUCKET" ? heldBucket : Reflect.get(target, prop),
    })
    return handler.fetch(request, heldEnv, ctx)
  },
}

export default worker

// @ts-ignore generated at build time
export { DOQueueHandler, DOShardedTagCache } from "./.open-next/worker.js"
```
A fuller file also passes each response through `withSharedCache(request, ...)`; that part is left out here.

## Verify

1. Request a new ISR or `"use cache"` page, twice, a few seconds apart.
2. After the first request, the entry is in the bucket. Wrangler has no `r2 object list`; look in the dashboard (R2, `your-app-next-cache`, filter by the build id), fetch a known key with `wrangler r2 object get your-app-next-cache/<key> --remote`, or list with an S3 client against the bucket (`rclone ls r2:your-app-next-cache --include '<build-id>/**'`).
3. The second request is served from cache instead of rendering again.

Measured on a slow dynamic page, three requests on a fresh build: without the wrapper 41.7, 31.0 and 30.8 s; with it 31.8 s (fills the cache), then 2.7 and 2.6 s. The entry that never persisted was its slowest `"use cache"` fill (up to five sequential API calls, 4 to 8 s each), because that put starts in the same tick the response body closes.

## Gotchas

- Symptom: cache entries missing from R2 although pages rendered. Cause: OpenNext does not await some puts and workerd cancels them with the response. Fix: the Proxy.
- Symptom: `Illegal invocation` from the bucket. Cause: native binding methods called with the Proxy as `this`. Fix: `value.bind(target)`.
- Symptom: `TypeError` on every request in an environment without the bucket. Cause: `new Proxy(undefined)`. Fix: the early return.
- First byte stayed about 2 s on one app after the fix: no per-URL route entry is written to R2 for a dynamic path, so each request resumes the build-time shell. Cause not found.
- The regional Cache API write (`withRegionalCache`) is not held by this and can still be cancelled. That cache is a front for R2, so a miss falls through to R2.
- `.catch(() => {})` only hides the failure from `waitUntil`. The caller of `put` still receives the rejection.
