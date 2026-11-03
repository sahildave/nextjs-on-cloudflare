---
title: Hyperdrive and Postgres on Cloudflare Workers
description: "Reach Supabase Postgres from a Next.js / Payload Worker through Hyperdrive, pinned next to the database, with a request-scoped pg pool that reads the binding per request."
tags: [cookbook, cloudflare]
topics: [cloudflare, hyperdrive, postgres, supabase, payload, pg, postgres-js]
date: 2026-11-03
path: 02-data/hyperdrive-supabase-postgres.md
---
# Hyperdrive and Postgres on Cloudflare Workers

You need this when a Next.js app on Workers talks to Postgres (Supabase, Neon, RDS) and uses `pg` (Payload's adapter) or `postgres.js`. A Worker cannot keep a TLS socket to Supabase's pooler, and workerd refuses to reuse a socket across requests. You get one connection string per request from the Hyperdrive binding, placement next to the database, and a `pg.Pool` subclass that opens a client per query.

This file owns the Hyperdrive config and the `pg` pool (`RequestScopedPool`). For the `postgres` (postgres.js) client, its `withSql` helper and its patch, see [./postgres-js-on-workers.md](./postgres-js-on-workers.md).

## Prerequisites

- Workers Paid plan (see [`plans-and-limits.md`](../01-start/plans-and-limits.md)). Free caps CPU at 10 ms and fails with error 1102.
- Next 16.3.8, `@opennextjs/cloudflare` 1.20.15, wrangler 4.156.x, `pg` 8.20.0, `postgres` 3.4.9, Payload 3.92.1. See [`nextjs-on-workers-opennext.md`](../01-start/nextjs-on-workers-opennext.md) for the base setup.
- `compatibility_flags: ["nodejs_compat"]` in `wrangler.jsonc`.
- A Postgres login whose password you can read from a secret store. Do not paste the password into a terminal; pipe it from your secret manager.

## Steps

1. Create the Hyperdrive config against the Supabase **session pooler** (port `5432`, not the transaction pooler on `6543`). Leave `?sslmode` params off the URL. Turn caching off so CMS edits show at once. Run `wrangler hyperdrive create --help` for the flag names on your wrangler version, including the one that sets the origin connection limit.

   ```bash
   wrangler hyperdrive create my-app-db \
     --connection-string="$(your-secret-manager print DATABASE_URI)" \
     --caching-disabled
   ```
2. Set the origin connection limit below the pooler's pool size. Hyperdrive defaults to 60; Supabase's session pooler takes 15, and above that Supavisor refuses with `EMAXCONNSESSION`. With a limit of 10, Hyperdrive queues instead of failing.
3. Bind it in `wrangler.jsonc` with a `localConnectionString`. `opennextjs-cloudflare deploy` and `preview` will not start without one.
4. Pin the Worker next to the database with `placement` (below). Check the `cf-placement` response header: `remote-FRA` means it worked.
5. Add `src/lib/db/hyperdrive.ts` and `src/lib/db/payload-pool.ts` from the Code section.
6. Pass `payloadPg` and `payloadPoolOptions` to Payload's Postgres adapter (`pg` and `pool` options, shown under Code), or use `payloadPg.Pool` wherever your code makes a `pg` pool.
7. If other code uses postgres.js, follow [./postgres-js-on-workers.md](./postgres-js-on-workers.md).
8. For a preview Worker, create a second config against the dev database and bind it under `env.preview` ([../06-operations/staging-and-preview-workers.md](../06-operations/staging-and-preview-workers.md)).
9. Run the tests and deploy.

## Code

### `wrangler.jsonc` (relevant part)

```jsonc
{
  "compatibility_date": "2026-10-01",
  "compatibility_flags": ["nodejs_compat", "global_fetch_strictly_public"],
  // Run next to the database, not next to the visitor: a page makes dozens
  // of queries, each a round trip. Unpinned, the Worker ran in Mumbai.
  "placement": { "region": "aws:eu-central-1" },
  "hyperdrive": [
    {
      "binding": "HYPERDRIVE",
      "id": "<HYPERDRIVE_ID>",
      // Local runs (preview, deploy's cache step) use a local Postgres.
      "localConnectionString": "postgresql://postgres:postgres@127.0.0.1:54322/postgres"
    }
  ]
}
```

Use the AWS region of your database (`aws:eu-central-1` for a Frankfurt Supabase).

### `src/lib/db/hyperdrive.ts`

```ts
import { getCloudflareContext } from "@opennextjs/cloudflare";

/** True inside workerd; false in Node (next build, scripts, tests). */
export const onWorkers = () =>
  typeof navigator !== "undefined" &&
  navigator.userAgent === "Cloudflare-Workers";

/**
 * The HYPERDRIVE binding's connection string, read per request. Payload and
 * `withSql` share it. Undefined outside a Worker request or when the binding
 * is missing.
 */
export function hyperdriveUrl(): string | undefined {
  if (!onWorkers()) return undefined;
  const env = getCloudflareContext().env as unknown as {
    HYPERDRIVE?: { connectionString: string };
  };
  return env.HYPERDRIVE?.connectionString;
}
```

### `src/lib/db/payload-pool.ts`

```ts
import pg from "pg";
import { hyperdriveUrl, onWorkers } from "./hyperdrive";

/**
 * Pool options for Payload's Postgres adapter. pg's default
 * `connectionTimeoutMillis` is 0, so a saturated pool leaves a render waiting
 * until the function is killed. A timeout makes it throw instead.
 * `max` caps this instance's share of the pooler: 5 per instance, 3 per build
 * worker, since the build prerenders on several workers at once.
 */
export const payloadPoolOptions = (
  connectionString: string | undefined,
): pg.PoolConfig => ({
  connectionString: connectionString ?? "",
  connectionTimeoutMillis: 10_000,
  max: process.env.NEXT_PHASE === "phase-production-build" ? 3 : 5,
});

/**
 * A Pool that opens a fresh client for every query or checkout and closes it
 * on release. workerd binds a socket to the request that opened it, so a
 * per-isolate pool that hands one request another request's client hangs or
 * throws "Cannot perform I/O on behalf of a different request". Pooling is
 * left to Hyperdrive. Drizzle checks `instanceof pg.Pool`, hence the subclass.
 */
export class RequestScopedPool extends pg.Pool {
  // biome-ignore lint/suspicious/noExplicitAny: pg's overloads
  override connect(callback?: any): any {
    const client = new pg.Client({
      ...this.options,
      connectionString: hyperdriveUrl() ?? this.options.connectionString,
    });
    // pg-cloudflare rejects its pending read when client.end() closes the
    // socket, and pg re-emits that as 'error'; with no listener it became an
    // unhandled rejection on every query. An error during a query still
    // rejects that query; one between queries (a socket reset in a checked-out
    // transaction) is logged here. After our own end() it is expected.
    let ended = false;
    const end = () => {
      ended = true;
      return client.end().catch(() => {});
    };
    client.on("error", (error) => {
      if (!ended) console.warn("payload pg client error", error);
    });
    const opened = client.connect().then(
      () => {
        (client as unknown as pg.PoolClient).release = () => {
          void end();
        };
        return client as unknown as pg.PoolClient;
      },
      // A failed connect can leave the socket open; end it before rethrowing.
      async (error) => {
        await end();
        throw error;
      },
    );
    if (!callback) return opened;
    opened.then(
      (c) => callback(undefined, c, c.release),
      (error) => callback(error),
    );
  }

  // biome-ignore lint/suspicious/noExplicitAny: pg's overloads
  override query(...args: any[]): any {
    const callback =
      typeof args[args.length - 1] === "function" ? args.pop() : undefined;
    const run = (async () => {
      const client = await this.connect();
      try {
        return await client.query(...(args as [string]));
      } finally {
        client.release();
      }
    })();
    if (!callback) return run;
    run.then((result) => callback(undefined, result), callback);
  }

  override end(): Promise<void> {
    return Promise.resolve();
  }
}

/** pg as Payload sees it: request-scoped on Workers, a normal pool in Node (build, scripts). */
export const payloadPg: typeof pg = onWorkers()
  ? { ...pg, Pool: RequestScopedPool }
  : pg;
```

Wire it in `payload.config.ts`:

```ts
db: postgresAdapter({
  schemaName: "payload",
  pg: payloadPg,
  pool: payloadPoolOptions(process.env.DATABASE_URI),
}),
```

### `scripts/payload-pool.test.ts` (three of five tests)

Run with `node --import tsx --test`.

```ts
import assert from "node:assert/strict";
import { type AddressInfo, createServer } from "node:net";
import { test } from "node:test";

import {
  payloadPg,
  payloadPoolOptions,
  RequestScopedPool,
} from "../src/lib/db/payload-pool";

// A server that accepts the socket and never answers: a pooler queueing the
// client. pg's default connectionTimeoutMillis (0) waits on it forever.
test("Payload's pool gives up on a connection that never answers", async () => {
  const sockets = new Set<import("node:net").Socket>();
  const server = createServer((socket) => sockets.add(socket));
  await new Promise<void>((done) => server.listen(0, "127.0.0.1", done));
  const { port } = server.address() as AddressInfo;
  const pool = new payloadPg.Pool(
    payloadPoolOptions(`postgresql://u:p@127.0.0.1:${port}/db`),
  );
  const started = Date.now();
  try {
    const outcome = await Promise.race([
      pool.connect().then(
        () => "connected",
        (error: Error) => error.message,
      ),
      new Promise((done) => setTimeout(() => done("still waiting"), 12_000)),
    ]);
    assert.match(String(outcome), /timeout/i);
    assert.ok(Date.now() - started < 11_000);
  } finally {
    for (const socket of sockets) socket.destroy();
    server.close();
    await pool.end().catch(() => {});
  }
});

// workerd binds a socket to the request that opened it: no client may be
// shared between two checkouts, and a released client must be closed.
test("RequestScopedPool opens a new client per checkout and ends it on release", async () => {
  const pool = new RequestScopedPool(
    payloadPoolOptions("postgresql://u:p@127.0.0.1:1/db"),
  );
  const opened: import("pg").Client[] = [];
  const ended: import("pg").Client[] = [];
  const pg = (await import("pg")).default;
  const realConnect = pg.Client.prototype.connect;
  const realEnd = pg.Client.prototype.end;
  pg.Client.prototype.connect = async function (this: import("pg").Client) {
    opened.push(this);
  } as unknown as typeof realConnect;
  pg.Client.prototype.end = async function (this: import("pg").Client) {
    ended.push(this);
  } as unknown as typeof realEnd;
  try {
    const a = await pool.connect();
    const b = await pool.connect();
    assert.notEqual(a, b);
    a.release();
    b.release();
    assert.deepEqual(ended, opened);
    assert.equal(opened.length, 2);
  } finally {
    pg.Client.prototype.connect = realConnect;
    pg.Client.prototype.end = realEnd;
  }
});

// pg-cloudflare rejects its pending read when client.end() closes the socket;
// with no 'error' listener it surfaced as an unhandled rejection per query.
test("RequestScopedPool gives each client an error listener before connect", async () => {
  const pool = new RequestScopedPool(
    payloadPoolOptions("postgresql://u:p@127.0.0.1:1/db"),
  );
  const listeners: number[] = [];
  const pg = (await import("pg")).default;
  const realConnect = pg.Client.prototype.connect;
  const realEnd = pg.Client.prototype.end;
  pg.Client.prototype.connect = async function (this: import("pg").Client) {
    listeners.push(this.listenerCount("error"));
  } as unknown as typeof realConnect;
  pg.Client.prototype.end = async () => {};
  try {
    const client = await pool.connect();
    assert.deepEqual(listeners, [1]);
    assert.doesNotThrow(() => client.emit("error", new Error("socket closed")));
    client.release();
  } finally {
    pg.Client.prototype.connect = realConnect;
    pg.Client.prototype.end = realEnd;
  }
});
```

The other two tests (a rejected `connect()` still ends its client; a client error warns unless our own `end()` caused it) follow the same stub pattern. The timeout test takes about 10 s.

## Verify

```bash
pnpm exec tsc --noEmit
node --test --import tsx scripts/payload-pool.test.ts
curl -sI https://your-domain.com/some-db-backed-page | grep -i cf-placement
# cf-placement: remote-FRA
wrangler tail --format=pretty   # no "Cannot perform I/O on behalf of a different request"
```

Fire 40 parallel requests at a database-backed page against `opennextjs-cloudflare preview`. A shared pool hangs under a burst, not on a single load.

## Gotchas

- **"Cannot perform I/O on behalf of a different request", or a hung isolate under a burst** → a per-isolate `pg.Pool` handed one request another's socket. `maxUses: 1` was not enough in other projects' tests under a 40-request burst. Use the request-scoped pool above and let Hyperdrive pool.
- **Payload reads its DB config once at startup, Hyperdrive's string is per request** → read the binding inside `connect()` (as `RequestScopedPool` does), not in `payload.config.ts`.
- **`EMAXCONNSESSION` from Supavisor** → Hyperdrive's `origin_connection_limit` defaults to 60 and the session pooler takes 15. Set it to 10.
- **"Connection terminated unexpectedly" on a direct TLS connection to the pooler** → Workers cannot hold that session. Go through Hyperdrive.
- **`unhandledRejection: This socket has been closed`, ~5k an hour** → a per-query `pg.Client` closed while pg-cloudflare's read was pending, and nobody listened for `'error'`. Add the listener before `connect()`.
- **Same error from `postgres.js` after `sql.end()`** → three unguarded `tcp.emit('error')` sites in its Workers polyfill. Patch: [./postgres-js-on-workers.md](./postgres-js-on-workers.md).
- **Deploy or preview refuses to start** → the Hyperdrive binding needs a `localConnectionString` in `wrangler.jsonc`.
- **Worker runs in Mumbai and every query crosses continents** → no `placement`. Pin it and check `cf-placement`.
- **Build prerender against the live DB timed out once** → retry before debugging. Keep `max` at 3 per build worker.
- **Shared-database migrations** stay manual and run before the code that needs them deploys.
- **Long synchronous CPU freezes the clock**, so pg's 10 s connect timeout can fire instantly afterwards. See [`payload-cms-on-workers.md`](../05-app-features/payload-cms-on-workers.md) for the PBKDF2 case and the one-tick fix.
