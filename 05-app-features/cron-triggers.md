---
title: Cron Triggers on Cloudflare Workers
description: "Run scheduled jobs with Worker Cron Triggers that call your existing /api/cron routes through a scheduled() handler, keeping the bearer check and the failure email."
tags: [cookbook, cloudflare]
topics: [cloudflare, cron, scheduled, workers]
date: 2026-11-03
path: 05-app-features/cron-triggers.md
---
# Cron Triggers on Cloudflare Workers

A common Next.js setup runs each scheduled job as a route under `/api/cron/*` that checks `Authorization: Bearer $CRON_SECRET`. On Workers you declare cron expressions in `wrangler.jsonc`, and a `scheduled()` handler turns each firing into the same request against the same route. The routes, their bearer check and their failure alerts do not change. You get three jobs (daily, weekly, hourly) running on the Worker.

## Prerequisites

- The Worker entry from [../01-start/nextjs-on-workers-opennext.md](../01-start/nextjs-on-workers-opennext.md).
- A `CRON_SECRET` runtime secret ([../01-start/secrets-and-env.md](../01-start/secrets-and-env.md)).
- Workers Paid. The free plan allows 5 cron triggers per account, and a deploy silently skips the rest.
- Existing routes under `/api/cron/*` that check the bearer.

## Steps

1. List your schedules in `wrangler.jsonc` under `triggers.crons`. Cloudflare cron syntax is standard five-field UTC.
2. Map each expression to its route in `worker.ts` (Code below). The expression string is the key, so write it identically in both files.
3. In `scheduled()`, build a `Request` to the route with the bearer header, call OpenNext's `handler.fetch`, and throw if the response is not ok. A thrown error marks the invocation failed in the dashboard.
4. Keep `CRON_SECRET` as a Worker secret. The routes keep rejecting every other caller (503 when the secret is unset, 401 on a wrong bearer).
5. Add `/api/cron` to your private path list so the Workers Cache never stores a cron response ([../03-caching/workers-cache-edge-caching.md](../03-caching/workers-cache-edge-caching.md)).
6. If another host still runs the same jobs, do not deploy the `triggers` block until the domain cutover, or each job runs twice. Merge it as part of the cutover, or leave it commented. Moving from Vercel: [../migrate-from-vercel/README.md](../migrate-from-vercel/README.md).
7. Set `limits.cpu_ms` in `wrangler.jsonc` if a job is CPU-heavy. Workers has no wall-clock cap for I/O-bound work, only CPU limits.

Cron CPU limits: 30 s if the schedule fires more often than hourly, 15 min if hourly or rarer. All three jobs here are hourly or rarer.

## Code

### `wrangler.jsonc` (excerpt)

```jsonc
// Each schedule maps to a /api/cron route in worker.ts.
"triggers": {
	"crons": ["0 4 * * *", "0 5 * * 1", "0 * * * *"]
},
"limits": { "cpu_ms": 300000 }
```

### `worker.ts`

`handler` is the OpenNext import from [../01-start/nextjs-on-workers-opennext.md](../01-start/nextjs-on-workers-opennext.md).

```ts
type Env = { CRON_SECRET?: string; WORKER_SELF_REFERENCE: Fetcher };

// Each Cron Trigger in wrangler.jsonc calls its route through the app, with
// the bearer the routes check (`cronUnauthorized`).
const CRON_ROUTES: Record<string, string> = {
  "0 4 * * *": "/api/cron/releases",
  "0 5 * * 1": "/api/cron/verify",
  "0 * * * *": "/api/cron/sponsor-holds",
};

export default {
  // async fetch(...) { ... }

  async scheduled(controller, env, ctx) {
    const path = CRON_ROUTES[controller.cron];
    if (!path) throw new Error(`No route for cron "${controller.cron}"`);
    const request = new Request(`https://cron.internal${path}`, {
      headers: { authorization: `Bearer ${env.CRON_SECRET ?? ""}` },
    });
    const response = await handler.fetch(request, env, ctx);
    if (!response.ok)
      throw new Error(`${path} answered ${response.status}`);
  },
} satisfies ExportedHandler<Env>;
```

`cron.internal` is never resolved. The request goes straight to the handler in the same isolate, so it also skips the `fetch` wrapper (no www redirect, no cache headers).

### `src/lib/cron-auth.ts`

```ts
import crypto from "node:crypto";

/**
 * Returns the answer to send when the request is not allowed (503 when the
 * secret is unset, 401 on a wrong bearer), or null to run.
 * The compare is constant time; a length mismatch fails.
 */
export const cronUnauthorized = (request: Request): Response | null => {
  const secret = process.env.CRON_SECRET;
  if (!secret)
    return Response.json({ code: "not_configured" }, { status: 503 });
  const given = Buffer.from(request.headers.get("authorization") ?? "");
  const expected = Buffer.from(`Bearer ${secret}`);
  const ok =
    given.length === expected.length && crypto.timingSafeEqual(given, expected);
  return ok ? null : Response.json({ code: "unauthorized" }, { status: 401 });
};
```

### A route (`src/app/api/cron/sponsor-holds/route.ts`, shape only)

```ts
import { withCronAlert } from "@/lib/cron-alert";
import { cronUnauthorized } from "@/lib/cron-auth";

export function GET(request: Request): Promise<Response> {
  return withCronAlert("sponsor-holds", () => run(request));
}

async function run(request: Request): Promise<Response> {
  const denied = cronUnauthorized(request);
  if (denied) return denied;
  // ...do the job; return a 5xx Response if any item failed
  return Response.json({ ok: true });
}
```

### `src/lib/cron-alert.ts`

`withCronAlert` emails you when a job throws or answers 5xx. A 401 sends nothing. `sendEmail` and `reportCaughtError` are the app's own mail and error-report helpers; swap in yours.

```ts
export const withCronAlert = async (
  cron: string,
  run: () => Promise<Response>,
): Promise<Response> => {
  let response: Response;
  try {
    response = await run();
  } catch (error) {
    reportCaughtError(error, "withCronAlert", { cron });
    await sendCronFailureAlert(
      cron,
      error instanceof Error ? (error.stack ?? error.message) : String(error),
    );
    return Response.json({ code: "cron_failed" }, { status: 500 });
  }
  if (response.status >= 500)
    await sendCronFailureAlert(
      cron,
      `HTTP ${response.status}: ${await response.clone().text()}`,
    );
  return response;
};
```

`sendCronFailureAlert` sends one email with subject `Cron failed: <name>`; a failed send is logged, never thrown. In the body, point at Cloudflare logs (Workers & Pages, your Worker, Logs).

### `scripts/cron-alert.test.ts` (cases)

The test runs under `node --experimental-test-module-mocks --import tsx --test`. It mocks `server-only` and the mail helper, then asserts:

```ts
test("a cron that throws answers 500 and emails the admin", async () => { /* status 500, 1 email, subject "Cron failed: releases" */ });
test("a cron that answers 5xx emails the admin and passes the response on", async () => { /* body kept, 1 email */ });
test("a successful or unauthorized run sends nothing", async () => { /* 200 and 401 -> 0 emails */ });
test("cronUnauthorized: 503 unset, 401 wrong or longer bearer, null when it matches", async () => {
  delete process.env.CRON_SECRET;
  assert.equal(cronUnauthorized(req("Bearer "))?.status, 503);
  process.env.CRON_SECRET = "s3cret";
  assert.equal(cronUnauthorized(req())?.status, 401);
  assert.equal(cronUnauthorized(req("Bearer nope00"))?.status, 401);
  assert.equal(cronUnauthorized(req("Bearer s3cretX"))?.status, 401);
  assert.equal(cronUnauthorized(req("Bearer s3cret")), null);
});
```

The full file is `scripts/cron-alert.test.ts`.

## Verify

Run a trigger locally, with the secret in `.dev.vars`:

```bash
echo 'CRON_SECRET=local-test' > .dev.vars
pnpm exec wrangler dev --test-scheduled
curl "http://localhost:8787/__scheduled?cron=0+*+*+*+*"
# the route runs; a non-ok answer shows as an error in the wrangler output
```

On the deployed Worker:

```bash
pnpm exec wrangler tail your-app --format pretty
# at the top of the hour, a "Scheduled" event for cron "0 * * * *" appears, outcome Ok
curl -s -o /dev/null -w "%{http_code}\n" https://your-domain.com/api/cron/releases   # 401 without the bearer
```

The dashboard (Worker, Settings, Triggers) lists the active schedules.

## Gotchas

- Jobs ran twice for a day → the old host's crons and the Worker triggers were both live → enable the Worker `triggers` only at cutover.
- Some cron triggers never fire after deploy, no error → free plan allows 5 per account across all Workers → use Workers Paid or consolidate schedules.
- `No route for cron "..."` → the expression in `triggers.crons` differs from the key in `CRON_ROUTES` by a character → copy-paste it.
- Route answers 401 from the scheduler → `CRON_SECRET` not pushed to the Worker → push it ([../01-start/secrets-and-env.md](../01-start/secrets-and-env.md)). A 503 means the secret is unset.
- A job that relied on a 300 s wall-clock cap elsewhere → no wall-clock limit applies to I/O on Workers, but CPU does → set `limits.cpu_ms` and keep batch size sane.
- Alert email points at the old host's logs → leftover text → point it at Workers Logs.
- A `/api/cron/*` response got cached → the path was missing from the private list → add it there and test it.
- One extra route may exist with no schedule (for example `/api/cron/newsletter`, called by hand). Leave unscheduled routes alone rather than inventing a trigger.
