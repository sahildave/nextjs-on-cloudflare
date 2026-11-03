---
title: Payload CMS 3 on Cloudflare Workers
description: "Run Payload 3 (admin at /cms, REST API, Postgres, R2 uploads) inside a Next.js Worker: the PBKDF2 login fix, the request-scoped pg pool wired in, a UI patch, R2 storage and cache-purging hooks."
tags: [cookbook, cloudflare]
topics: [cloudflare, workers, payload, cms, postgres, hyperdrive, r2, pbkdf2, opennext]
date: 2026-11-03
path: 05-app-features/payload-cms-on-workers.md
---
# Payload CMS 3 on Cloudflare Workers

Payload 3 runs inside your Next.js app, so on OpenNext it runs inside the Worker. It works, but four things break on the way: the `pg` pool, password hashing at login, image processing, and Payload's own client-schema cache. This recipe has the Payload side of each fix and the config for R2 uploads. The pool code itself lives in [../02-data/hyperdrive-supabase-postgres.md](../02-data/hyperdrive-supabase-postgres.md), the image code in [./images-without-sharp.md](./images-without-sharp.md), and the publish hooks in [../03-caching/cache-invalidation-and-purge.md](../03-caching/cache-invalidation-and-purge.md).

## Prerequisites

- Next.js on Workers with OpenNext and Workers Paid: [../01-start/nextjs-on-workers-opennext.md](../01-start/nextjs-on-workers-opennext.md). Versions: Next 16.3.8, `@opennextjs/cloudflare` 1.20.15, wrangler 4.156.x, Payload 3.92.1 (`payload`, `@payloadcms/next`, `@payloadcms/db-postgres`, `@payloadcms/richtext-lexical`, `@payloadcms/storage-s3`, `@payloadcms/ui`), `pg` 8.20.0, `@noble/hashes` 2.4.0.
- A Hyperdrive binding named `HYPERDRIVE` in front of Postgres: [../02-data/hyperdrive-supabase-postgres.md](../02-data/hyperdrive-supabase-postgres.md).
- WASM image processing if you process uploads: [./images-without-sharp.md](./images-without-sharp.md).
- Edge cache and purge helpers for the revalidate hooks: [../03-caching/workers-cache-edge-caching.md](../03-caching/workers-cache-edge-caching.md), [../03-caching/cache-invalidation-and-purge.md](../03-caching/cache-invalidation-and-purge.md).
- Secrets (`PAYLOAD_SECRET`, `DATABASE_URI`, R2 keys) pushed from your secret store: [../01-start/secrets-and-env.md](../01-start/secrets-and-env.md).

## Steps

1. Route layout. Put Payload's admin at `/cms` and its REST API at `/api/cms`, so your own `/admin` and `/api/*` stay free. Files under Code: `src/app/(payload)/layout.tsx`, `src/app/(payload)/cms/[[...segments]]/page.tsx`, `src/app/(payload)/api/cms/[...slug]/route.ts`.
2. Wrap `next.config.mjs` with `withPayload`.
3. Wire the database: `postgresAdapter` with `pg: payloadPg` and `pool: payloadPoolOptions(process.env.DATABASE_URI)`. `payloadPg` swaps in `RequestScopedPool` on Workers (one client per query, closed on release) and stays a normal pool in Node (build, scripts). Copy `src/lib/db/hyperdrive.ts` and `src/lib/db/payload-pool.ts` from [../02-data/hyperdrive-supabase-postgres.md](../02-data/hyperdrive-supabase-postgres.md).
4. Add the PBKDF2 fallback. Create `src/lib/pbkdf2-fallback.ts`, then call `installPbkdf2Fallback()` once at the top of `worker.ts`.
5. Apply the `@payloadcms/ui` patch with pnpm (`patches/` + `patchedDependencies`).
6. Uploads: use `s3Storage` against R2's S3 endpoint. Do not set Payload's `sharp` option; process images before `payload.create` ([./images-without-sharp.md](./images-without-sharp.md)).
7. Keep the admin and API out of the shared cache: leave `/cms` and `/api/` off the shared allowlist in your Worker cache wrapper (with the denylist variant, add them to `PRIVATE_PREFIXES`), and mark `/cms(/.*)?` and `/api/cms(/.*)?` public in your own auth gate (Payload checks its own users).
8. Add `afterChange` and `afterDelete` hooks that expire your cache tags and purge Workers Cache: `revalidateHooks` in [../03-caching/cache-invalidation-and-purge.md](../03-caching/cache-invalidation-and-purge.md). A direct `payload.db.*` write skips them; call `expireTagsAndPurge` after it.
9. Run Payload migrations from a machine, not from the Worker (see Gotchas). Run them before deploying code that needs them.
10. Deploy and log in once with `wrangler dev --remote`, because only the real runtime enforces the PBKDF2 cap.

## Code

### `next.config.mjs`

```js
import { withPayload } from "@payloadcms/next/withPayload";

/** @type {import('next').NextConfig} */
const nextConfig = {
  images: { unoptimized: true },
  // ...your config
};

export default withPayload(nextConfig);
```

### `src/payload.config.ts`

R2 through the S3 API. A bucket with an EU jurisdiction needs the `.eu.` endpoint; the plain one answers 403 on every call. Without all the env vars (a placeholder build) the plugin is disabled and Payload keeps local disk storage, but `alwaysInsertFields: true` keeps the plugin's columns in the schema so generated migrations stay stable.

```ts
import path from "node:path";
import { fileURLToPath } from "node:url";
import { postgresAdapter } from "@payloadcms/db-postgres";
import { lexicalEditor } from "@payloadcms/richtext-lexical";
import { s3Storage } from "@payloadcms/storage-s3";
import { buildConfig } from "payload";
import { Media } from "./collections/Media";
import { Users } from "./collections/Users";
import { payloadPg, payloadPoolOptions } from "./lib/db/payload-pool";
import { payloadSecret } from "./payload-env";

const dirname = path.dirname(fileURLToPath(import.meta.url));

const r2 = {
  accountId: process.env.R2_ACCOUNT_ID,
  accessKeyId: process.env.R2_ACCESS_KEY_ID,
  secretAccessKey: process.env.R2_SECRET_ACCESS_KEY,
  bucket: process.env.R2_BUCKET,
  cdnUrl: process.env.MEDIA_CDN_URL?.replace(/\/$/, ""),
};

export default buildConfig({
  routes: { admin: "/cms", api: "/api/cms" },
  admin: {
    user: Users.slug,
    importMap: { baseDir: path.resolve(dirname) },
  },
  collections: [Media, Users],
  editor: lexicalEditor(),
  secret: payloadSecret(), // throws when PAYLOAD_SECRET is empty
  typescript: { outputFile: path.resolve(dirname, "payload-types.ts") },
  db: postgresAdapter({
    schemaName: "payload", // keep Payload in its own schema on a shared database
    push: false, // schema changes only through migrations
    pg: payloadPg,
    pool: payloadPoolOptions(process.env.DATABASE_URI),
  }),
  plugins: [
    s3Storage({
      enabled: Object.values(r2).every(Boolean),
      alwaysInsertFields: true,
      collections: {
        media: {
          disablePayloadAccessControl: true,
          generateFileURL: ({ filename, prefix }) =>
            `${r2.cdnUrl}/${prefix ? `${prefix}/` : ""}${filename}`,
        },
      },
      bucket: r2.bucket ?? "",
      config: {
        endpoint: `https://${r2.accountId}.eu.r2.cloudflarestorage.com`, // drop ".eu" for the default jurisdiction
        region: "auto",
        forcePathStyle: true,
        credentials: {
          accessKeyId: r2.accessKeyId ?? "",
          secretAccessKey: r2.secretAccessKey ?? "",
        },
      },
    }),
  ],
});
```

There is no `sharp` key. Payload must never sign with an empty secret, so a one-line helper throws:

```ts
// src/payload-env.ts
export const payloadSecret = (
  env: Record<string, string | undefined> = process.env,
): string => {
  const secret = env.PAYLOAD_SECRET;
  if (!secret) throw new Error("PAYLOAD_SECRET is not set");
  return secret;
};
```

### `src/collections/Media.ts`

No `imageSizes`, no `crop`, no `focalPoint`: those need `sharp`.

```ts
import type { CollectionConfig } from "payload";

export const Media: CollectionConfig = {
  slug: "media",
  upload: { mimeTypes: ["image/*"] },
  access: {
    read: () => true,
    create: ({ req }) => Boolean(req.user),
    update: ({ req }) => Boolean(req.user),
    delete: ({ req }) => Boolean(req.user),
  },
  fields: [{ name: "alt", type: "text", required: true }],
};
```

### `src/app/(payload)/layout.tsx`

```tsx
import config from "@payload-config";
import "@payloadcms/next/css";
import { RootLayout, handleServerFunctions } from "@payloadcms/next/layouts";
import type { ServerFunctionClient } from "payload";
import type { ReactNode } from "react";
import { importMap } from "./cms/importMap.js";

const serverFunction: ServerFunctionClient = async (args) => {
  "use server";
  return handleServerFunctions({ ...args, config, importMap });
};

export default function Layout({ children }: { children: ReactNode }) {
  return (
    <RootLayout
      config={config}
      importMap={importMap}
      serverFunction={serverFunction}
    >
      {children}
    </RootLayout>
  );
}
```

### `src/app/(payload)/cms/[[...segments]]/page.tsx`

```tsx
import config from "@payload-config";
import { RootPage, generatePageMetadata } from "@payloadcms/next/views";
import type { Metadata } from "next";
import { importMap } from "../importMap.js";

type Args = {
  params: Promise<{ segments: string[] }>;
  searchParams: Promise<{ [key: string]: string | string[] }>;
};

export const generateMetadata = ({
  params,
  searchParams,
}: Args): Promise<Metadata> =>
  generatePageMetadata({ config, params, searchParams });

export default function Page({ params, searchParams }: Args) {
  return RootPage({ config, params, searchParams, importMap });
}
```

Regenerate `importMap.js` (`payload generate:importmap`) whenever you add a field or editor feature that has an admin component.

### `src/app/(payload)/api/cms/[...slug]/route.ts`

```ts
import config from "@payload-config";
import {
  REST_DELETE,
  REST_GET,
  REST_OPTIONS,
  REST_PATCH,
  REST_POST,
  REST_PUT,
} from "@payloadcms/next/routes";

export const GET = REST_GET(config);
export const POST = REST_POST(config);
export const DELETE = REST_DELETE(config);
export const PATCH = REST_PATCH(config);
export const PUT = REST_PUT(config);
export const OPTIONS = REST_OPTIONS(config);
```

### `src/lib/db/payload-pool.ts`

The `payloadPg` and `payloadPoolOptions` imported above, with `RequestScopedPool` and its tests, are in [../02-data/hyperdrive-supabase-postgres.md](../02-data/hyperdrive-supabase-postgres.md). On Workers `payloadPg` swaps in a pool that opens one client per query from the per-request Hyperdrive string; in Node it is plain `pg`.

### `src/lib/pbkdf2-fallback.ts`

```ts
import nodeCrypto from "node:crypto";
import { pbkdf2Async } from "@noble/hashes/pbkdf2.js";
import { sha256 } from "@noble/hashes/sha2.js";

// A deployed Worker refuses PBKDF2 above 100,000 iterations ("Pbkdf2 failed: iteration
// counts above 100000 are not supported"). The cap is observed, not documented: the
// hosted runtime sets it, and local workerd has none. Payload hashes /cms passwords with
// crypto.pbkdf2(sha256) at 600,000, and its login swallows the error, so every
// login on Workers answered "The email or password provided is incorrect."
export const WORKERD_PBKDF2_MAX_ITERATIONS = 100_000;

type Pbkdf2Callback = (err: Error | null, derivedKey: Buffer) => void;
type Pbkdf2 = (
  password: string | Uint8Array,
  salt: string | Uint8Array,
  iterations: number,
  keylen: number,
  digest: string,
  callback: Pbkdf2Callback,
) => void;

/** PBKDF2-HMAC-SHA256 in JS; byte-identical to Node's crypto.pbkdf2. */
export async function pbkdf2Sha256(
  password: string | Uint8Array,
  salt: string | Uint8Array,
  iterations: number,
  keylen: number,
): Promise<Buffer> {
  const key = await pbkdf2Async(sha256, password, salt, {
    c: iterations,
    dkLen: keylen,
  });
  return Buffer.from(key.buffer, key.byteOffset, key.byteLength);
}

/**
 * Route crypto.pbkdf2(sha256) calls above workerd's cap to the JS version.
 * Payload reads `crypto.pbkdf2` from the shared `node:crypto` default export
 * at call time, so replacing the property once at Worker start covers it.
 */
export function installPbkdf2Fallback(crypto = nodeCrypto): void {
  const native = crypto.pbkdf2 as unknown as Pbkdf2;
  const patched: Pbkdf2 = (
    password,
    salt,
    iterations,
    keylen,
    digest,
    callback,
  ) => {
    if (iterations <= WORKERD_PBKDF2_MAX_ITERATIONS || digest !== "sha256") {
      native(password, salt, iterations, keylen, digest, callback);
      return;
    }
    // workerd freezes Date.now() and timers while synchronous JS runs, so
    // after ~15 s of hashing the clock is stale. Login's next pg connect then
    // sees its 10 s connectionTimeoutMillis already spent and fails with
    // "Connection terminated unexpectedly". One timer tick refreshes the clock.
    const tick = () => new Promise((resolve) => setTimeout(resolve, 0));
    pbkdf2Sha256(password, salt, iterations, keylen).then(
      async (key) => {
        await tick();
        callback(null, key);
      },
      async (err: Error) => {
        await tick();
        callback(err, Buffer.alloc(0));
      },
    );
  };
  (crypto as unknown as { pbkdf2: Pbkdf2 }).pbkdf2 = patched;
}
```

### `worker.ts` (the relevant lines)

```ts
// @ts-ignore `.open-next/worker.js` exists only after a build; worker.ts is excluded from tsconfig
import { default as handler } from "./.open-next/worker.js";
import { installPbkdf2Fallback } from "./src/lib/pbkdf2-fallback";

// Payload's /cms login and password set hash at 600,000 PBKDF2 iterations,
// over the Workers cap. Replace crypto.pbkdf2 once, at Worker start.
installPbkdf2Fallback();

export default {
  fetch: (request, env, ctx) => handler.fetch(request, env, ctx),
} satisfies ExportedHandler;

// @ts-ignore generated at build time
export { DOQueueHandler, DOShardedTagCache } from "./.open-next/worker.js";
```

### `patches/@payloadcms__ui@3.92.1.patch`

Payload caches each collection's client field map per server process, keyed by slug only. An anonymous form-state request (login, forgot-password) builds the map from the logged-out config, which has no fields, and that empty map then serves every signed-in render. Applies payloadcms/payload#18168 (open at the time; issue #18162). Drop it once a Payload release includes the fix.

```diff
diff --git a/dist/utilities/getClientSchemaMap.js b/dist/utilities/getClientSchemaMap.js
index e42de04dd6d85d8876b1b30ac471fb71b2f3f973..f41f777d1012bba1d1be342f8e2c0edb90e1484c 100644
--- a/dist/utilities/getClientSchemaMap.js
+++ b/dist/utilities/getClientSchemaMap.js
@@ -17,7 +17,8 @@ export const getClientSchemaMap = cache(args => {
   if (!cachedClientSchemaMap || global._payload_doNotCacheClientSchemaMap) {
     cachedClientSchemaMap = new Map();
   }
-  const cacheKey = collectionSlug || globalSlug || `widget:${widgetSlug}`;
+  // Unauthenticated configs omit field schemas, so they cannot share a cache entry (payloadcms/payload#18168).
+  const cacheKey = `${config.unauthenticated ? "unauthenticated" : "authenticated"}:${collectionSlug || globalSlug || `widget:${widgetSlug}`}`;
   let cachedEntityClientFieldMap = cachedClientSchemaMap.get(cacheKey);
   if (cachedEntityClientFieldMap) {
     return cachedEntityClientFieldMap;
```

`pnpm-workspace.yaml`:

```yaml
patchedDependencies:
  '@payloadcms/ui@3.92.1': patches/@payloadcms__ui@3.92.1.patch
```

Create it on your version with `pnpm patch @payloadcms/ui`, edit the same line, then `pnpm patch-commit`. This bug is not Workers-specific, but a long-lived isolate hits it sooner than a short-lived function.

### `scripts/pbkdf2-fallback.test.ts`

```ts
import assert from "node:assert/strict";
import crypto from "node:crypto";
import { describe, test } from "node:test";

import {
  WORKERD_PBKDF2_MAX_ITERATIONS,
  installPbkdf2Fallback,
  pbkdf2Sha256,
} from "../src/lib/pbkdf2-fallback";

// Payload's current (600,000 / 32) and legacy (25,000 / 512) parameters.
const PASSWORD = "correct horse battery staple";
const SALT = crypto.randomBytes(32).toString("hex");

describe("pbkdf2Sha256", () => {
  for (const [iterations, keylen] of [
    [600_000, 32],
    [25_000, 512],
  ]) {
    test(`matches crypto.pbkdf2 at ${iterations} iterations, ${keylen} bytes`, async () => {
      const expected = crypto.pbkdf2Sync(
        PASSWORD,
        SALT,
        iterations,
        keylen,
        "sha256",
      );
      const actual = await pbkdf2Sha256(PASSWORD, SALT, iterations, keylen);
      assert.equal(actual.toString("hex"), expected.toString("hex"));
    });
  }
});

describe("installPbkdf2Fallback", () => {
  // A stand-in for workerd's node:crypto, which throws above the cap.
  function workerdCrypto() {
    const calls: number[] = [];
    const fake = {
      pbkdf2(
        password: string,
        salt: string,
        iterations: number,
        keylen: number,
        digest: string,
        callback: (err: Error | null, key?: Buffer) => void,
      ) {
        calls.push(iterations);
        if (iterations > WORKERD_PBKDF2_MAX_ITERATIONS)
          throw new RangeError(
            `Pbkdf2 failed: iteration counts above ${WORKERD_PBKDF2_MAX_ITERATIONS} are not supported (requested ${iterations}).`,
          );
        crypto.pbkdf2(password, salt, iterations, keylen, digest, callback);
      },
    };
    return { fake, calls };
  }

  const hash = (
    c: { pbkdf2: typeof crypto.pbkdf2 },
    iterations: number,
    keylen: number,
  ) =>
    new Promise<string>((resolve, reject) =>
      c.pbkdf2(PASSWORD, SALT, iterations, keylen, "sha256", (err, key) =>
        err ? reject(err) : resolve(key.toString("hex")),
      ),
    );

  test("answers above the cap with Node's bytes", async () => {
    const { fake, calls } = workerdCrypto();
    const c = fake as unknown as typeof crypto;
    const over = WORKERD_PBKDF2_MAX_ITERATIONS + 1;
    await assert.rejects(hash(c, over, 32), /iteration counts above 100000/);

    installPbkdf2Fallback(c);
    assert.equal(
      await hash(c, over, 32),
      crypto.pbkdf2Sync(PASSWORD, SALT, over, 32, "sha256").toString("hex"),
    );
    assert.deepEqual(calls, [over]); // the patched call never reached native
  });

  test("keeps the native call at or under the cap", async () => {
    const { fake, calls } = workerdCrypto();
    const c = fake as unknown as typeof crypto;
    installPbkdf2Fallback(c);
    assert.equal(
      await hash(c, 25_000, 512),
      crypto.pbkdf2Sync(PASSWORD, SALT, 25_000, 512, "sha256").toString("hex"),
    );
    assert.deepEqual(calls, [25_000]);
  });

  // workerd freezes Date.now() and timers while synchronous JS runs, so the
  // ~15 s of hashing leaves the clock stale; pg's connectionTimeoutMillis then
  // fires at once on login's next connect. A timer tick before the callback
  // lets the clock catch up. A timer scheduled before the call must fire first.
  // Date.now() is frozen as on workerd, so noble's asyncLoop never yields.
  test("waits one timer tick between the key and the callback", async (t) => {
    const { fake } = workerdCrypto();
    const c = fake as unknown as typeof crypto;
    installPbkdf2Fallback(c);
    const now = Date.now();
    t.mock.method(Date, "now", () => now);
    const order: string[] = [];
    setTimeout(() => order.push("timer"), 0);
    await new Promise<void>((resolve, reject) =>
      c.pbkdf2(
        PASSWORD,
        SALT,
        WORKERD_PBKDF2_MAX_ITERATIONS + 1,
        32,
        "sha256",
        (err) => {
          order.push("callback");
          err ? reject(err) : resolve();
        },
      ),
    );
    assert.deepEqual(order, ["timer", "callback"]);
  });
});
```

## Verify

1. Offline tests:
   ```bash
   node --import tsx --test scripts/pbkdf2-fallback.test.ts
   ```
   Expect 5 passing. Run the pool tests from [../02-data/hyperdrive-supabase-postgres.md](../02-data/hyperdrive-supabase-postgres.md) as well.
2. The real runtime. Local workerd has no PBKDF2 cap, so a local login passes even without the fix. Use remote:
   ```bash
   pnpm exec wrangler dev --remote
   ```
   Open `/cms`, sign in with an account created before the migration (600,000-iteration hash). Expect the admin to load. Login costs about 15 s of CPU, so the account needs the `limits.cpu_ms` raised on the Worker (300000 works).
3. After deploy, tail the Worker and publish a doc:
   ```bash
   pnpm exec wrangler tail
   ```
   Expect no `unhandledRejection`, no "Cannot perform I/O on behalf of a different request", and no `edge cache purge skipped`.
4. Public page after publish:
   ```bash
   curl -sI https://your-domain.com/ | grep -i cf-cache-status
   ```
   Expect `MISS` on the first request after the publish, then `HIT` on the next.
5. Upload an image in `/cms`; the media URL should start with your `MEDIA_CDN_URL` and return `200` with a WebP content type if you convert on the server.

## Gotchas

- Symptom: `/cms` login says "email or password incorrect" for a correct password. Cause: Payload 3 hashes at 600,000 PBKDF2 iterations, a deployed Worker caps PBKDF2 at 100,000 (observed, not documented), and Payload swallows the error. Fix: wrap `crypto.pbkdf2` and send calls over the cap to `@noble/hashes` (byte-identical to Node). Local workerd and miniflare have no cap, so only `wrangler dev --remote` or a real deploy shows it. Cost: about 15 s of CPU per login.
- Symptom: after that fix, login fails with "cannot begin transaction: Connection terminated unexpectedly" or Payload's generic "Something went wrong" toast. Cause: workerd freezes `Date.now()` and timers during synchronous CPU, so after 15 s of hashing pg's 10 s connect timeout looks already spent. Fix: await one `setTimeout(0)` before the callback, as `installPbkdf2Fallback` does.
- Symptom: "Cannot perform I/O on behalf of a different request", a hang under a burst, `unhandledRejection: This socket has been closed` on every query, or EMAXCONNSESSION from Supabase. Cause: a per-isolate `pg.Pool`, a missing `'error'` listener, or Hyperdrive's connection limit above the pooler's. Cloudflare's own Payload post suggests `maxUses: 1`; a later port (sourceplane/lumen-payload#9) found a shared pool still hung under 40 parallel requests. Fix: `RequestScopedPool` and the limits in [../02-data/hyperdrive-supabase-postgres.md](../02-data/hyperdrive-supabase-postgres.md).
- Symptom: Payload reads `DATABASE_URI` once, but Hyperdrive's string is per request. Fix: `RequestScopedPool` reads `hyperdriveUrl()` inside `connect()`. Keep `DATABASE_URI` set anyway for build and scripts.
- Symptom: `/cms/account` or `/cms/collections/users/create` renders only its header, with `TypeError: Cannot use 'in' operator to search for 'hidden' in undefined` in your error tracker. Cause: Payload's client-schema cache is keyed by slug and an anonymous request (login form) fills it with the field-less logged-out map. Fix: the `@payloadcms/ui` patch above.
- Symptom: a CMS publish does not change the public page. Cause: Workers Cache and Next's data cache both hold it, and `revalidateTag` only queues the tag. Fix: the hooks in [../03-caching/cache-invalidation-and-purge.md](../03-caching/cache-invalidation-and-purge.md). A direct `payload.db.*` write skips hooks entirely, so call the helper after it (claiming an app with a pending draft left the old developer on the public page).
- Symptom: Payload CLI (`payload migrate`) cannot load the config. Cause: the config imports `src/` code that tsx loads as CommonJS. Fix: load the config and migrations from ESM copies in a script of your own (`scripts/payload-migrate.mts`). Whatever you do, run migrations from a machine against the real `DATABASE_URI`, with `NODE_ENV=production` so the adapter never pushes a schema diff on connect, and on the session pooler (`:5432`), not the transaction pooler (`:6543`).
- Shared-database migrations stay manual and owner-only. Run them before the code that needs them deploys: merging `main` can bring in a migration the Worker depends on.
- Bundle size: Next plus Payload is about 50 MB uncompressed against a 64 MiB limit, and a cold isolate must load it and boot Payload (fresh-instance cache misses took 3 to 4 s). Splitting the CMS into its own Worker is the fix considered, not tried here.
- Source maps: `upload_source_maps` stays `false`; the maps are over the 60 MiB API cap.
- A `payload.create` with an image: it passes through `s3Storage` to R2. If the S3 call 403s, check the endpoint's jurisdiction segment (`.eu.`) first.
- Payload's default logger uses `pino-pretty`, reported to need Node APIs Workers lack. A config that sets no logger still boots on Workers, so nothing to do here. If you see a pino error at boot, pass a console logger in `buildConfig`.
