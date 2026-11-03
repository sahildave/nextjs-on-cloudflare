---
title: Cache invalidation and purge on Cloudflare Workers
description: "Make a CMS publish show on the next request: revalidateTag, wait for the tag write, purge Workers Cache, purge again after the regional tag-cache window, and keep OpenNext's tag-cache Durable Objects next to the Worker."
tags: [cookbook, cloudflare]
topics: [cloudflare, cache-invalidation, workers-cache, purge, revalidateTag, durable-objects, opennext, payload]
date: 2026-11-03
path: 03-caching/cache-invalidation-and-purge.md
---
# Cache invalidation and purge on Cloudflare Workers

You need this when public pages sit in Workers Cache (`./workers-cache-edge-caching.md`) and an editor, a webhook or a cron changes the data. The recipe expires Next cache tags, waits until the tag write has landed, purges Workers Cache, purges a second time once the regional tag cache has expired, and warms the home page. It also moves OpenNext's tag-cache Durable Objects next to the Worker, which cut tag lookups from 158 ms to 16 ms.

## Prerequisites

- Next 16.3.8 with `cacheComponents`, `@opennextjs/cloudflare` 1.20.15, wrangler 4.156.x. See [`nextjs-on-workers-opennext.md`](../01-start/nextjs-on-workers-opennext.md) and `./cache-components-ppr-on-workers.md`.
- Workers Cache on (`"cache": { "enabled": true }`) and the headers from `./workers-cache-edge-caching.md`.
- Bindings: `NEXT_INC_CACHE_R2_BUCKET` (R2), `NEXT_CACHE_DO_QUEUE` and `NEXT_TAG_CACHE_DO_SHARDED` (Durable Objects), and a self service binding `WORKER_SELF_REFERENCE`.
- The tests use `node:test` with `mock.module`, which needs `--experimental-test-module-mocks` on Node versions where it is still flagged. The recipe was built on Node 24.

## Steps

1. Configure OpenNext's caches in `open-next.config.ts` (below): R2 incremental cache behind the regional cache, the DO queue, and the sharded tag cache pinned to `weur` with a 5 s regional cache.
2. Declare the bindings, DO classes and migration in `wrangler.jsonc`, and re-export `DOQueueHandler` and `DOShardedTagCache` from `worker.ts`.
3. Add `src/lib/tag-cache.ts` and `src/lib/edge-cache.ts`.
4. Route every write through `expireTagsAndPurge(tags)`. In Payload, do it from `afterChange` and `afterDelete` hooks. After a direct `payload.db.*` write, which skips those hooks, call it yourself.
5. Tag the cached reads. Every public read carries one shared tag (`catalog`); a single-doc read adds its own (`app:<slug>`).
6. Run the tests, deploy, publish a change, and run Verify.

Order inside `expireTagsAndPurge` matters:

1. `revalidateTag` for each tag. This only queues the tag.
2. Await Next's pending revalidation writes (`executeRevalidates`).
3. `ctx.cache.purge({ purgeEverything: true })`.
4. Purge again 7 s later through `waitUntil`, then warm `/` through the self binding.

A purge that runs before the tag write lets the next request re-render from the stale data cache and put the old page back.

## Code

### `wrangler.jsonc` (relevant part)

```jsonc
{
  "cache": { "enabled": true },
  "r2_buckets": [
    { "binding": "NEXT_INC_CACHE_R2_BUCKET", "bucket_name": "my-app-next-cache" }
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
  // The Worker's binding to itself, for warming the home page.
  "services": [{ "binding": "WORKER_SELF_REFERENCE", "service": "my-app" }]
}
```

### `worker.ts` (relevant part)

```ts
// @ts-expect-error `.open-next/worker.js` is generated at build time
export { DOQueueHandler, DOShardedTagCache } from "./.open-next/worker.js";
```

### `open-next.config.ts`

```ts
import { defineCloudflareConfig } from "@opennextjs/cloudflare";
import r2IncrementalCache from "@opennextjs/cloudflare/overrides/incremental-cache/r2-incremental-cache";
import { withRegionalCache } from "@opennextjs/cloudflare/overrides/incremental-cache/regional-cache";
import doQueue from "@opennextjs/cloudflare/overrides/queue/do-queue";
import doShardedTagCache from "@opennextjs/cloudflare/overrides/tag-cache/do-sharded-tag-cache";
import { TAG_CACHE_REGIONAL_TTL_SEC } from "./src/lib/tag-cache";

// Each data center caches tag revalidation times for a few seconds, so a page
// stops asking a Durable Object for every tag on every cache miss. Tags never
// written are cached too ("missing tags"): almost every tag a page reads (the
// `_N_T_/` path tags) is never written, and without this they would always miss.
const tagCache = doShardedTagCache({
  baseShardSize: 12,
  regionalCache: true,
  regionalCacheTtlSec: TAG_CACHE_REGIONAL_TTL_SEC,
  regionalCacheDangerouslyPersistMissingTags: true,
});

// Pin the tag-cache Durable Objects to western Europe, next to the Worker
// (placement aws:eu-central-1). 1.20.15 sets a location hint only through
// regional replication, which picks a region from the visitor's continent, so
// every shard id gets the "weur" region here instead: the region joins the
// object's name (`…;region-weur`), creating new objects, and `get()` passes it
// as the location hint. A hint applies only when an object is created, which
// is why the name has to change.
// biome-ignore lint/complexity/useLiteralKeys: the method is TS-private
const generateDOIdArray = tagCache["generateDOIdArray"].bind(tagCache);
// biome-ignore lint/complexity/useLiteralKeys: as above
tagCache["generateDOIdArray"] = (options) =>
  generateDOIdArray(options).map((entry) => {
    entry.doId.region = "weur";
    return entry;
  });

// "use cache", ISR and revalidateTag share one cache across every isolate:
// pages and data in R2 (fronted by the colo cache), tags in sharded Durable
// Objects, time-based revalidation deduplicated by the DO queue.
export default defineCloudflareConfig({
  incrementalCache: withRegionalCache(r2IncrementalCache, {
    mode: "long-lived",
  }),
  tagCache,
  queue: doQueue,
});
```

Change `weur` to the region next to your database (`enam`, `apac`, and so on). It is a Durable Object location hint.

### `src/lib/tag-cache.ts`

```ts
/**
 * How long each data center's Cache API keeps a tag's revalidation time in
 * front of the tag-cache Durable Objects (`regionalCacheTtlSec` in
 * open-next.config.ts). A tag write clears the entry only in the data center
 * that wrote it, so elsewhere a fresh invalidation can stay hidden this long;
 * `purgeEdgeCache` purges a second time after it for that reason.
 */
export const TAG_CACHE_REGIONAL_TTL_SEC = 5;
```

### `src/lib/edge-cache.ts`

```ts
import { getCloudflareContext } from "@opennextjs/cloudflare";
import { revalidateTag } from "next/cache";
import { workAsyncStorage } from "next/dist/server/app-render/work-async-storage.external";
import { executeRevalidates } from "next/dist/server/revalidation-utils";
import { TAG_CACHE_REGIONAL_TTL_SEC } from "./tag-cache";

/** Every public read carries CATALOG_TAG. */
export const CATALOG_TAG = "catalog";

/** The Worker's binding to itself (`WORKER_SELF_REFERENCE` in wrangler.jsonc). */
type SelfBinding = { fetch(request: Request): Promise<Response> };

type PurgeableContext = {
  waitUntil(promise: Promise<unknown>): void;
  cache?: { purge(options: { purgeEverything: true }): Promise<unknown> };
};

/**
 * Drops every page Workers Cache holds for this Worker, so a change shows on
 * the next request. Writes are rare, so one purge-everything beats tracking
 * which pages list which record. The purge outlives the response through
 * `waitUntil`. Outside a Worker (next dev, scripts) it does nothing; inside one
 * without the purge API it warns, since Workers Cache then keeps the stale
 * page until its `s-maxage` runs out.
 */
export function purgeEdgeCache(): void {
  let ctx: PurgeableContext;
  let env: { WORKER_SELF_REFERENCE?: SelfBinding };
  try {
    const context = getCloudflareContext();
    ctx = context.ctx as unknown as PurgeableContext;
    env = context.env as { WORKER_SELF_REFERENCE?: SelfBinding };
  } catch {
    return;
  }
  if (!ctx.cache) {
    console.warn(
      "edge cache purge skipped: ctx.cache is missing (is cache.enabled set in wrangler.jsonc?)",
    );
    return;
  }
  const { cache } = ctx;
  const purge = () =>
    cache
      .purge({ purgeEverything: true })
      .catch((error) => console.warn("edge cache purge failed", error));
  ctx.waitUntil(purge());
  // Another data center can read the tag's old time from its own cache for up
  // to TAG_CACHE_REGIONAL_TTL_SEC after the write, serve the old page and put
  // it back in Workers Cache. Purging again once that window has passed drops
  // it. Best effort, about 7 s.
  ctx.waitUntil(
    new Promise((done) => setTimeout(done, REPURGE_AFTER_MS))
      .then(purge)
      .then(() => warmHome(env.WORKER_SELF_REFERENCE)),
  );
}

/**
 * Renders `/` into Workers Cache once the second purge is done, so the first
 * visitor after a write does not wait for the render (about 8 s). The request
 * goes through the Worker's own service binding: Workers Cache sits in front
 * of every `fetch()` invocation, binding calls included, and keys on path, not
 * host. A `fetch` to the site's own origin from inside the Worker would not
 * come back through it. `/` varies on Accept, so the warm sends a browser's
 * navigation Accept. Best effort: a failure only warns.
 */
async function warmHome(self: SelfBinding | undefined): Promise<void> {
  try {
    if (!self) throw new Error("WORKER_SELF_REFERENCE is not bound");
    const response = await self.fetch(
      new Request(
        new URL("/", process.env.NEXT_PUBLIC_APP_URL || "https://your-domain.com"),
        {
          headers: { accept: BROWSER_ACCEPT },
          signal: AbortSignal.timeout(WARM_TIMEOUT_MS),
        },
      ),
    );
    // Reading the body lets the cache store the whole response.
    await response.arrayBuffer();
    if (!response.ok) throw new Error(`/ answered ${response.status}`);
  } catch (error) {
    console.warn("home page warm failed", error);
  }
}

const WARM_TIMEOUT_MS = 20_000;

/** Chrome's Accept on a page navigation, the most common visitor variant. */
const BROWSER_ACCEPT =
  "text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7";

/** The regional tag-cache window plus a margin for the write's own latency. */
export const REPURGE_AFTER_MS = (TAG_CACHE_REGIONAL_TTL_SEC + 2) * 1000;

/**
 * Expires `tags`, waits for the tag writes, then purges Workers Cache.
 * `revalidateTag` only queues a tag; Next writes it after the action or route
 * handler returns. A purge that ran first would let the next request
 * re-render from the stale data cache and put the old page back, so the queue
 * is flushed here first. Next flushes it again at the end of the request;
 * writing the same tag twice is harmless. Throws outside a Next request, as
 * `revalidateTag` does.
 *
 * The shared tag is only marked stale ("max"): every shell carries it, so
 * expiring it made the next visitor to each page wait for a full rebuild
 * (34 s on `/`). Stale, the old page is served while one background render
 * refreshes it. Every other tag, the written doc's own included, expires at
 * once. `removed` (a doc unpublished or deleted) expires the shared tag at once
 * too, so the doc leaves every listing on the next request.
 */
export async function expireTagsAndPurge(
  tags: Iterable<string>,
  { removed = false }: { removed?: boolean } = {},
): Promise<void> {
  for (const tag of tags)
    revalidateTag(tag, tag === CATALOG_TAG && !removed ? "max" : { expire: 0 });
  const store = workAsyncStorage.getStore();
  if (store) await executeRevalidates(store);
  purgeEdgeCache();
}
```

`next/dist/...` imports are private Next paths. Re-check them on every Next upgrade; the test below fails loudly if the order breaks.

### Payload hooks that call it (`src/collections/hooks/revalidate.ts`, trimmed)

```ts
import type {
  CollectionAfterChangeHook,
  CollectionAfterDeleteHook,
  PayloadRequest,
} from "payload";
import { CATALOG_TAG, expireTagsAndPurge } from "../../lib/edge-cache";

type Doc = { id?: number | string; slug?: string | null; _status?: string };
type TagsOf<T extends Doc> = (doc: T, req: PayloadRequest) => Promise<string[]>;

// A collection without drafts has no `_status`: every saved doc is live.
const isLive = (doc: Doc | undefined) =>
  doc?.id != null && (doc._status === undefined || doc._status === "published");

/**
 * Expires `catalog` plus the doc's tags; `removed` when a doc left the site.
 * Outside Next (the Payload CLI, the import scripts) `revalidateTag` has no
 * request scope and throws; the write must not fail for it.
 */
const expire = async <T extends Doc>(
  tagsOf: TagsOf<T>,
  docs: T[],
  req: PayloadRequest,
  removed = false,
) => {
  try {
    const tags = new Set([CATALOG_TAG]);
    for (const doc of docs)
      for (const tag of await tagsOf(doc, req)) tags.add(tag);
    await expireTagsAndPurge(tags, { removed });
  } catch (error) {
    console.warn("revalidate skipped", error);
  }
};

/**
 * afterChange and afterDelete hooks that expire a collection's cached public
 * reads. They fire when the doc is published now or was before (publish,
 * edit, unpublish), never for a draft-only save, and never with
 * `context.disableRevalidate` (the importers).
 */
export const revalidateHooks = <T extends Doc>(tagsOf: TagsOf<T>) => {
  const afterChange: CollectionAfterChangeHook<T & { id: number }> = async ({
    doc,
    previousDoc,
    req,
  }) => {
    if (req.context.disableRevalidate) return doc;
    // Both versions' tags: a rename expires the old slug too.
    const docs = [doc, previousDoc].filter((d) => d?.id != null);
    if (docs.some(isLive))
      await expire(tagsOf, docs, req, isLive(previousDoc) && !isLive(doc));
    return doc;
  };
  const afterDelete: CollectionAfterDeleteHook<T & { id: number }> = async ({
    doc,
    req,
  }) => {
    if (!req.context.disableRevalidate && isLive(doc))
      await expire(tagsOf, [doc], req, true);
  };
  return { afterChange: [afterChange], afterDelete: [afterDelete] };
};
```

Use it per collection: `hooks: revalidateHooks(async (doc) => ["app:" + doc.slug])`. Return early on `disableRevalidate` before any dynamic import: importers load a Payload config copy that is deleted once it connects, so a lazy import in a hook fails every import write.

### `scripts/edge-cache.test.ts` (trimmed)

```ts
import assert from "node:assert/strict";
import { afterEach, mock, test } from "node:test";

// The second, delayed purge waits on setTimeout; these tests move the clock.
mock.timers.enable({ apis: ["setTimeout"] });

const log: string[] = [];
let ctx: {
  waitUntil(p: Promise<unknown>): void;
  cache?: { purge(o: unknown): Promise<unknown> };
};
const withPurge = () => ({
  waitUntil: (p: Promise<unknown>) => void p,
  cache: { purge: async () => void log.push("purge") },
});
ctx = withPurge();
let env: Record<string, unknown> = {};

mock.module("@opennextjs/cloudflare", {
  namedExports: { getCloudflareContext: () => ({ env, ctx }) },
});
mock.module("next/cache", {
  namedExports: {
    revalidateTag: (tag: string, profile: unknown) =>
      log.push(`revalidate ${tag} ${JSON.stringify(profile)}`),
  },
});
mock.module("next/dist/server/app-render/work-async-storage.external", {
  namedExports: { workAsyncStorage: { getStore: () => ({}) } },
});
mock.module("next/dist/server/revalidation-utils", {
  namedExports: {
    executeRevalidates: () =>
      new Promise<void>((done) =>
        setImmediate(() => {
          log.push("tags written");
          done();
        }),
      ),
  },
});

afterEach(() => {
  mock.timers.reset();
  mock.timers.enable({ apis: ["setTimeout"] });
  log.length = 0;
  ctx = withPurge();
  env = {};
});

test("tags are written before Workers Cache is purged", async () => {
  const { expireTagsAndPurge } = await import("../src/lib/edge-cache");
  await expireTagsAndPurge(["catalog", "app:foo"]);
  assert.deepEqual(log, [
    'revalidate catalog "max"',
    'revalidate app:foo {"expire":0}',
    "tags written",
    "purge",
  ]);
});

test("a removed doc expires catalog at once", async () => {
  const { expireTagsAndPurge } = await import("../src/lib/edge-cache");
  await expireTagsAndPurge(["catalog"], { removed: true });
  assert.equal(log[0], 'revalidate catalog {"expire":0}');
});

test("a Worker without the purge API warns", async (t) => {
  const warn = t.mock.method(console, "warn", () => {});
  ctx = { waitUntil: () => {} };
  const { purgeEdgeCache } = await import("../src/lib/edge-cache");
  purgeEdgeCache();
  assert.equal(warn.mock.callCount(), 1);
  assert.match(String(warn.mock.calls[0].arguments[0]), /ctx\.cache is missing/);
});

test("the edge cache is purged again after the regional tag-cache window", async () => {
  const pending: Promise<unknown>[] = [];
  ctx = { ...withPurge(), waitUntil: (p) => void pending.push(p) };
  const { purgeEdgeCache, REPURGE_AFTER_MS } = await import("../src/lib/edge-cache");
  purgeEdgeCache();
  await pending[0];
  assert.deepEqual(log, ["purge"]);
  mock.timers.tick(REPURGE_AFTER_MS - 1);
  await new Promise(setImmediate);
  assert.deepEqual(log, ["purge"]);
  mock.timers.tick(1);
  await pending[1];
  assert.deepEqual(log, ["purge", "purge"]);
});
```

The full file also tests the Payload hooks (publish marks `catalog` stale and expires `app:<slug>`; unpublish and delete expire it at once) and the home warm (one GET through the self binding with a `text/html` Accept, a failed warm only warns).

### `scripts/tag-cache.test.ts` (what it checks)

It stubs the Durable Object namespace and the Cache API, imports `open-next.config`, resolves the tag cache override, and asserts:

- every read and write names an object ending `;region-weur` and passes `locationHint: "weur"`, and reads and writes name the same objects (a write to an object no read asks would never show);
- a second `getLastRevalidated` is served from the data center's cache, never-written tags included, and a write clears it.

Keep a test like this. It is what tells you after an OpenNext upgrade that the private `generateDOIdArray` override stopped applying.

## Verify

```bash
# 1. Prime the cache.
curl -s -o /dev/null https://your-domain.com/apps/some-app
curl -sI https://your-domain.com/apps/some-app | grep -i cf-cache-status   # HIT

# 2. Publish a change in the CMS (or call your write path), then:
curl -s https://your-domain.com/apps/some-app | grep -c "new tagline"      # 1 on the next request
curl -sI https://your-domain.com/apps/some-app | grep -i cf-cache-status   # MISS, then HIT

# 3. Seven seconds later the second purge and the home warm have run.
wrangler tail --format=pretty   # no "edge cache purge skipped", no "home page warm failed"
curl -sI https://your-domain.com/ | grep -i cf-cache-status                # HIT after the warm
```

Tag lookups: before the pinning, each `getTagData` cost ~158 ms with 3.2 lookups per request. After: 16 ms median, 1.5 lookups.

## Gotchas

- **Old page comes back after a purge** → the purge ran before the tag write, so the next request re-rendered from the stale data cache. Await `executeRevalidates` first.
- **A write from another data center is hidden for up to 5 s** → the regional tag cache. The second purge at 7 s closes that for pages. A render that started before the write and finishes after the second purge, or a failed purge, can still leave a stale page cached until `s-maxage`.
- **A direct `payload.db.*` write skips Payload's revalidate hooks** → call `expireTagsAndPurge` after it. Claiming an app with a pending draft left the old developer on the public page until we did.
- **Expiring the shared tag made `/` take 34 s for the next visitor** → `catalog` is on every shell. Mark it stale (`"max"`), expire it with `{ expire: 0 }` only when a doc leaves the site.
- **First visitor after a write waits ~8 s on `/`** → `/` is a PPR page that refreshes inside a visitor's request. Warm it through `WORKER_SELF_REFERENCE`. A `fetch` to the public origin from inside the Worker does not pass through Workers Cache.
- **Tag lookups at 158 ms** → the Durable Objects were created far from the Worker (probably by a cron, which is not placed). 1.20.15 has no public `locationHint`, so override `generateDOIdArray`. Don't use `shardReplication.regionalReplication`: it keys on the visitor's continent.
- **The location hint applies only when an object is created** → changing it needs new object names (the `;region-weur` suffix). Objects without the suffix keep the old tags. Expect tags written before the change to be forgotten.
- **Purge warns "ctx.cache is missing"** → `cache.enabled` is not set in `wrangler.jsonc`, or the runtime is old.
- **After a deploy** nothing needs purging: Workers Cache is partitioned by Worker version. The tag-cache Durable Objects are not, which is why a rollback does not clear an expired tag. See [`incident-playbook.md`](../06-operations/incident-playbook.md).
- **Re-check on upgrades:** the `generateDOIdArray` override, `next/dist/server/revalidation-utils` and `work-async-storage.external` are private. The tests catch a change.
- **Importers and CLIs** → `revalidateTag` throws outside a Next request. Catch it in the hook and set `req.context.disableRevalidate` for imports.
- **A count on a cached page goes stale in other data centers after a write** → OpenNext's regional tag cache clears only the colo that wrote. One app hit this with a pre-order count on `/` and dropped the regional tag cache: `doShardedTagCache({ baseShardSize: 12 })` with no `regionalCache`. Another keeps it on and covers the 5 s window with the second purge. Pick one.
