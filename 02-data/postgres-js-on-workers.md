---
title: postgres.js over Hyperdrive on Cloudflare Workers
description: "Use the postgres.js client from a Worker through a Hyperdrive binding, with one short-lived client per call (withSql) and a patch for the unhandled-rejection noise on sql.end()."
tags: [cookbook, cloudflare]
topics: [cloudflare, hyperdrive, postgres, postgres-js, supabase]
date: 2026-11-03
path: 02-data/postgres-js-on-workers.md
---
# postgres.js over Hyperdrive on Cloudflare Workers

You need this when server code in a Next.js app on Workers talks to Postgres with `postgres` (postgres.js) instead of `pg` or an ORM. You get a `withSql` wrapper that opens one client per call from the Hyperdrive binding and closes it, and the `postgres@3.4.9` patch that stops workerd from logging an unhandled rejection after `sql.end()`.

This file owns `withSql` and the postgres.js patch. The Hyperdrive config, placement, and the `pg` pool Payload uses are in [./hyperdrive-supabase-postgres.md](./hyperdrive-supabase-postgres.md).

## Prerequisites

- Hyperdrive config and binding: [./hyperdrive-supabase-postgres.md](./hyperdrive-supabase-postgres.md).
- `@opennextjs/cloudflare` 1.20.15, wrangler 4.156.x, `postgres` 3.4.9, Next 16.
- `compatibility_flags: ["nodejs_compat"]` in `wrangler.jsonc`.
- pnpm (the patch uses `patchedDependencies`; npm and yarn have their own patch tools).

## Steps

1. Add the binding to `wrangler.jsonc` (see Code). `localConnectionString` is what `wrangler dev`, `preview` and OpenNext's deploy step use.
2. Add `src/lib/db/hyperdrive.ts` and `src/lib/db/with-sql.ts`.
3. Add `postgres` to `serverExternalPackages` in `next.config.mjs` so the workerd build resolves unbundled.
4. Save the patch as `patches/postgres@3.4.9.patch` and register it in `pnpm-workspace.yaml`. Run `pnpm install`.
5. Run every query through `withSql`. Never keep a client at module scope.
6. Drop the patch when postgres.js ships its own fix. Check `cf/polyfills.js` in the new release for guarded `tcp.emit('error')` calls.

## Code

### `wrangler.jsonc` (excerpt)
```jsonc
{
  "compatibility_flags": ["nodejs_compat"],
  "hyperdrive": [
    {
      "binding": "HYPERDRIVE",
      "id": "<HYPERDRIVE_ID>",
      "localConnectionString": "postgresql://postgres:postgres@127.0.0.1:54322/postgres"
    }
  ]
}
```

### `src/lib/db/hyperdrive.ts`

The same file as in [./hyperdrive-supabase-postgres.md](./hyperdrive-supabase-postgres.md); repeated so this recipe stands alone.

```ts
import { getCloudflareContext } from "@opennextjs/cloudflare";

/** True inside workerd; false in Node (next build, scripts, tests). */
export const onWorkers = () =>
  typeof navigator !== "undefined" &&
  navigator.userAgent === "Cloudflare-Workers";

/**
 * The HYPERDRIVE binding's connection string, read per request. Undefined
 * outside a Worker request or when the binding is missing.
 */
export function hyperdriveUrl(): string | undefined {
  if (!onWorkers()) return undefined;
  const env = getCloudflareContext().env as unknown as {
    HYPERDRIVE?: { connectionString: string };
  };
  return env.HYPERDRIVE?.connectionString;
}
```

### `src/lib/db/with-sql.ts`

```ts
import { hyperdriveUrl, onWorkers } from "@/lib/db/hyperdrive";
import postgres from "postgres";

/**
 * The connection string: the HYPERDRIVE binding on Workers, DATABASE_URL in
 * Node. On Workers a missing binding throws: a direct pooler connection is not
 * an option there.
 */
function runtimeDatabaseUrl(): string {
  if (onWorkers()) {
    const url = hyperdriveUrl();
    if (!url) throw new Error("The HYPERDRIVE binding is not set");
    return url;
  }
  const url = process.env.DATABASE_URL;
  if (!url) throw new Error("DATABASE_URL is not set");
  return url;
}

/**
 * One connection per call, closed in `finally` (serverless: max 1, no prepared
 * statements). `fetch_types: false` skips the pg_type catalog read postgres.js
 * runs on every new connection: ~15k of them returned 6.7M rows of pooler
 * egress. No caller binds an array parameter, which is what it is for.
 */
export async function withSql<T>(
  run: (sql: postgres.Sql) => Promise<T>,
): Promise<T> {
  const sql = postgres(runtimeDatabaseUrl(), {
    max: 1,
    prepare: false,
    fetch_types: false,
  });
  try {
    return await run(sql);
  } finally {
    await sql.end();
  }
}
```

Usage:

```ts
const rows = await withSql((sql) => sql`select id from items where slug = ${slug}`);
```

### `patches/postgres@3.4.9.patch`

The real file from a production app. It guards all three `tcp.emit('error', …)` sites in the Workers socket polyfill: STARTTLS, connect, and the socket's `error()` function.

```diff
diff --git a/cf/polyfills.js b/cf/polyfills.js
index 53c5203dd5b057d741d14d708ba83b27751b527c..c988dc15baa9c14f2e7daff5de5236baa8237b91 100644
--- a/cf/polyfills.js
+++ b/cf/polyfills.js
@@ -116,7 +116,8 @@ export const tls = {
     tcp.raw = tcp.raw.startTls({ servername })
     tcp.raw.closed.then(
       () => tcp.emit('close'),
-      (e) => tcp.emit('error', e)
+      // Unguarded, a rejection after sql.end() dropped the listeners throws.
+      (e) => tcp.listenerCount('error') && tcp.emit('error', e)
     )
     tcp.writer = tcp.raw.writable.getWriter()
     tcp.reader = tcp.raw.readable.getReader()
@@ -155,7 +156,8 @@ function Socket() {
             ? close()
             : ((tcp.readyState = 'open'), tcp.emit('secureConnect'))
         },
-        (e) => tcp.emit('error', e)
+        // Unguarded, a rejection after sql.end() dropped the listeners throws.
+        (e) => tcp.listenerCount('error') && tcp.emit('error', e)
       )
       tcp.writer = tcp.raw.writable.getWriter()
       tcp.reader = tcp.raw.readable.getReader()
@@ -211,7 +213,9 @@ function Socket() {
   }
 
   function error(err) {
-    tcp.emit('error', err)
+    // A read that fails after postgres dropped its listeners (sql.end()) would
+    // throw from emit() and surface as an unhandled rejection.
+    tcp.listenerCount('error') && tcp.emit('error', err)
     tcp.emit('close')
   }
 }
```

To make it yourself: `pnpm patch postgres@3.4.9`, edit the three lines in `cf/polyfills.js`, `pnpm patch-commit <dir>`.

### `pnpm-workspace.yaml` (excerpt)
```yaml
patchedDependencies:
  postgres@3.4.9: patches/postgres@3.4.9.patch
```

### `scripts/with-sql.test.ts`

Run with `node --experimental-test-module-mocks --import tsx --test scripts/with-sql.test.ts`. postgres.js opens no socket until a query, so `sql.options` shows which host `withSql` chose without a database.

```ts
import assert from "node:assert/strict";
import { afterEach, mock, test } from "node:test";

let env: Record<string, { connectionString: string }> = {};
mock.module("@opennextjs/cloudflare", {
  namedExports: { getCloudflareContext: () => ({ env, ctx: {} }) },
});
const opened: string[] = [];
const login = async (sql: import("postgres").Sql) => {
  opened.push(`${sql.options.user}@${sql.options.host[0]}`);
};

const realNavigator = Object.getOwnPropertyDescriptor(globalThis, "navigator");
const onWorkers = (yes: boolean) =>
  Object.defineProperty(globalThis, "navigator", {
    configurable: true,
    value: { userAgent: yes ? "Cloudflare-Workers" : "Node.js" },
  });
afterEach(() => {
  if (realNavigator)
    Object.defineProperty(globalThis, "navigator", realNavigator);
  opened.length = 0;
  env = {};
  Reflect.deleteProperty(process.env, "DATABASE_URL");
});

const withSql: typeof import("../src/lib/db/with-sql").withSql = async (run) =>
  (await import("../src/lib/db/with-sql")).withSql(run);

test("on Workers withSql connects through HYPERDRIVE", async () => {
  onWorkers(true);
  env = { HYPERDRIVE: { connectionString: "postgres://app@hd/db" } };
  process.env.DATABASE_URL = "postgres://app@pooler/db";
  await withSql(login);
  assert.deepEqual(opened, ["app@hd"]);
});

test("on Workers without HYPERDRIVE withSql throws instead of dialing the pooler", async () => {
  onWorkers(true);
  process.env.DATABASE_URL = "postgres://app@pooler/db";
  await assert.rejects(withSql(login), /HYPERDRIVE binding is not set/);
  assert.deepEqual(opened, []);
});

test("in Node withSql uses DATABASE_URL", async () => {
  onWorkers(false);
  process.env.DATABASE_URL = "postgres://app@pooler/db";
  await withSql(login);
  assert.deepEqual(opened, ["app@pooler"]);
});

// The polyfill imports `cloudflare:sockets`, so Node cannot run it; read the
// installed file and check every error emit is guarded.
test("the Workers socket polyfill never emits 'error' with no listener", async () => {
  const { readFileSync } = await import("node:fs");
  const source = readFileSync(
    new URL("../node_modules/postgres/cf/polyfills.js", import.meta.url),
    "utf8",
  );
  const emits = source.match(/.*tcp\.emit\('error'.*/g) ?? [];
  assert.ok(emits.length >= 3, `found ${emits.length} error emits`);
  for (const line of emits)
    assert.match(line, /tcp\.listenerCount\('error'\) &&/, line.trim());
});
```

## Verify

1. `node --experimental-test-module-mocks --import tsx --test scripts/with-sql.test.ts` passes 4 tests. Delete the patch and reinstall: the polyfill test fails on the first unguarded line.
2. Deploy, then `wrangler tail <worker>` while you hit a route that queries through `withSql`.
3. Expected: the route returns 200 and the tail shows no `This socket has been closed.` unhandled rejection.

## Gotchas

- Symptom: `Unhandled rejection: This socket has been closed.` once per connection, in `wrangler tail`. Cause: `sql.end()` drops the socket listeners, then a late error calls `tcp.emit('error')` with no listener. Node tolerates it; workerd reports it. Fix: the patch. Patching only the `error()` site is not enough; the STARTTLS and connect sites need the same guard.
- Symptom: the binding is undefined at module load. Cause: a Worker has no env until a request runs. Fix: call `getCloudflareContext()` inside the request, as `hyperdriveUrl()` does. Never cache the result at module scope.
- Outside a Worker (`next build`, tests, scripts) `getCloudflareContext()` throws. The `onWorkers()` guard avoids it.
- `fetch_types: false` stops postgres.js from binding array parameters. If you bind arrays, turn it back on and accept one catalog read per connection.
- `max: 1, prepare: false`: one socket per call, no named prepared statements. Hyperdrive does the pooling, so a shared client pool saves nothing on Workers.
- A missing binding: the `withSql` above throws. A softer version returns `undefined` instead, and its callers do nothing (a sign-in count) or answer 503 (checkout). Decide that behaviour on purpose.
