---
title: Observability and logs on Cloudflare Workers
description: "Turn on Workers Logs, tail production, check placement and cache headers, and send server errors to PostHog with the right environment and release tags, without letting preview deploys file production issues."
tags: [cookbook, cloudflare]
topics: [cloudflare, observability, logs, wrangler-tail, posthog, error-tracking]
date: 2026-11-03
path: 06-operations/observability-and-logs.md
---
# Observability and logs on Cloudflare Workers

On Workers you get Workers Logs (turned on in `wrangler.jsonc`), `wrangler tail` for live output, response headers for placement and cache state, and whatever error tracker you wire in yourself. This recipe covers all four, using PostHog as the tracker.

## Prerequisites

- A Worker per [../01-start/nextjs-on-workers-opennext.md](../01-start/nextjs-on-workers-opennext.md), deployed with `APP_ENV` and `GIT_SHA` per [../01-start/deploy-from-github-actions.md](../01-start/deploy-from-github-actions.md).
- For error tracking: `posthog-node` and `NEXT_PUBLIC_POSTHOG_KEY`.
- Cache and incident headers are covered in [../03-caching/workers-cache-edge-caching.md](../03-caching/workers-cache-edge-caching.md) and [./incident-playbook.md](./incident-playbook.md).

## Steps

1. Enable Workers Logs in `wrangler.jsonc`: `"observability": { "enabled": true }`. Read them in the dashboard under Workers & Pages, your Worker, Logs. If `next.config` sets `compiler.removeConsole`, exclude `error` and `warn`, or your own server warnings never reach these logs (Gotchas).
2. Tail live traffic when debugging a deploy (Verify below).
3. Pin and verify placement. Add `"placement": { "region": "aws:eu-central-1" }` (use your database's AWS region), then look for `cf-placement: remote-FRA` on a response. Unpinned, one Worker ran in Mumbai and every query crossed continents.
4. Make environment explicit. `wrangler.jsonc` defaults `"vars": { "APP_ENV": "preview" }`, and only the main-branch deploy passes `--var=APP_ENV:production`. The error tracker filters its alerts and emails on `environment = production`, so a hand deploy or a preview cannot file production issues. Also use a separate `APP_ENV=preview` value on any test Worker.
5. Tag every error with `release: GIT_SHA`, so an error ties to the commit that threw it.
6. Report server errors from `instrumentation.ts` (`onRequestError`) and report caught errors from your catch blocks with a helper (Code below).
7. Send events after the response. In a Next request, use `after()` from `next/server`; on Workers it keeps the isolate alive through `waitUntil`. Never `await` a capture before a redirect or API response (the `/d` and `/go` redirects waited on PostHog per click before this was fixed).
8. Upload source maps to the tracker, not to Cloudflare. Cloudflare's upload cap is 60 MiB and a Next + Payload app's maps exceeded it, so `"upload_source_maps": false` and upload them to PostHog instead.
9. Alert on failed cron jobs by email: see [../05-app-features/cron-triggers.md](../05-app-features/cron-triggers.md).

## Code

### `wrangler.jsonc` (excerpt)

```jsonc
"observability": { "enabled": true },
"upload_source_maps": false,
"vars": { "APP_ENV": "preview" },
"placement": { "region": "aws:eu-central-1" }
```

### `src/instrumentation.ts`

Next 16 never calls `onRequestError` for proxy errors. Report those from `proxy.ts` yourself.

```ts
import type { Instrumentation } from "next";

// `request.path` is not sent: it can carry a query string.
export const onRequestError: Instrumentation.onRequestError = async (
  error,
  request,
  context,
) => {
  if (process.env.NEXT_RUNTIME !== "nodejs") return;
  const { reportServerError } = await import("@/lib/posthog/server");
  await reportServerError(error, {
    route: context.routePath,
    route_type: context.routeType,
    method: request.method,
  });
};
```

### `src/lib/posthog/server.ts` (the error path)

`captureExceptionImmediate` sends inside the call, so the event leaves before the isolate ends. One attempt, a 3 s ceiling, and it never throws.

```ts
import "server-only";
import { PostHog } from "posthog-node";

const RELEASE = process.env.GIT_SHA;
let client: PostHog | null | undefined;

function posthogServer(): PostHog | null {
  if (client !== undefined) return client;
  const key = process.env.NEXT_PUBLIC_POSTHOG_KEY?.trim();
  client = key
    ? new PostHog(key, {
        host: "https://eu.i.posthog.com", // direct: no ad blocker in front of a server
        requestTimeout: 3000,
        fetchRetryCount: 0, // the default retries 3x, holding a failed request open
        disableGeoip: true,
      })
    : null;
  return client;
}

export async function reportServerError(
  error: unknown,
  context: Record<string, unknown> = {},
): Promise<void> {
  try {
    const posthog = posthogServer();
    if (!posthog) return;
    await posthog.captureExceptionImmediate(error, undefined, {
      ...context,
      environment: process.env.APP_ENV ?? "development",
      release: RELEASE,
      $process_person_profile: false,
    });
  } catch {
    // Observability must never affect the response.
  }
}
```

A production filter also drops Next's control-flow throws (redirect and not-found digests), Node stream-shutdown errors, and scrubs events before send. Add your own filter for framework noise.

### `src/lib/after-response.ts`

```ts
import { after } from "next/server";

type Schedule = (task: () => Promise<unknown>) => void;

/**
 * Runs `task` after the response is sent when called inside a Next request,
 * so the platform keeps it alive without holding the response open. Outside a
 * request scope `after()` throws (importer scripts, the Payload CLI), and the
 * task is awaited instead.
 */
export async function runAfterResponse(
  task: () => Promise<unknown>,
  schedule: Schedule = after,
): Promise<void> {
  try {
    schedule(task);
  } catch {
    await task();
  }
}
```

### `src/lib/posthog/caught.ts`

```ts
import { runAfterResponse } from "@/lib/after-response";

/**
 * A caught error that still matters (the catch answers the user, so Next never
 * sees it). Keeps the console line and sends `$exception` after the response.
 * `context` holds safe ids only, never tokens, emails or user input.
 */
export function reportCaughtError(
  error: unknown,
  where: string,
  context: Record<string, string | number | boolean | null | undefined> = {},
): void {
  console.error(where, error);
  void runAfterResponse(async () => {
    const { reportServerError } = await import("@/lib/posthog/server");
    await reportServerError(error, { ...context, where, caught: true });
  }).catch(() => {});
}
```

The lazy `import()` keeps every module that calls the helper importable from offline tests.

## Verify

```bash
# Live logs for the production Worker
pnpm exec wrangler tail your-app --format pretty
pnpm exec wrangler tail your-app --status error      # only failed invocations

# Placement and cache state
curl -sI https://your-domain.com/ | grep -iE 'cf-placement|cf-cache-status'
# cf-placement: remote-FRA
# cf-cache-status: HIT        (MISS on the first request after a deploy or purge)
```

Fire the debug route (for example `/api/debug/posthog-error`, behind an admin check, or run it once and delete it) and confirm an issue appears in PostHog with `environment=production` and `release=<sha>`. From a preview deploy, the same call must show `environment=preview` and raise no alert.

## Gotchas

- A preview or hand deploy files production issues → `APP_ENV` was `production` outside the main job → keep the default `preview` in `wrangler.jsonc` and pass `production` only from the main-branch deploy.
- A fix is not live and `wrangler tail` shows old behavior → `src/` code compiles into the Next bundle, `worker.ts` does not → rebuild with `opennextjs-cloudflare build`, then deploy, and confirm the new version in `tail`.
- Error events never arrive → the function ended before the send finished → use `captureExceptionImmediate` with a short timeout, or send inside `after()`. Do not rely on a background flush timer.
- A capture slowed redirects → it was awaited before the response → move it into `after()`.
- Noisy `unhandledRejection: This socket has been closed` (about 5k per hour) → a per-query `pg.Client` without an `'error'` listener → see [../02-data/hyperdrive-supabase-postgres.md](../02-data/hyperdrive-supabase-postgres.md).
- "Worker's code had hung" or 524 on every page → see [./incident-playbook.md](./incident-playbook.md). Rolling back does not clear it when the cause is an expired tag in the tag-cache Durable Object.
- Proxy errors do not reach `onRequestError` → Next 16 skips it for `proxy.ts` → report from the proxy itself.
- The dashboard shows an "unexpected deploy" after `wrangler secret` changes → each secret write makes a new version → expected.
- **Your own `console.warn` and `console.error` lines never reach Workers Logs** → `compiler.removeConsole: true` strips them from server code too → use `removeConsole: { exclude: ["error", "warn"] }` (or drop the option). Checked on an app whose `next.config.mjs` sets `removeConsole: process.env.NODE_ENV === "production"`:
  - Next 16's docs (`node_modules/next/dist/docs/03-architecture/nextjs-compiler.md`) say the transform removes all `console.*` calls in application code (not `node_modules`), and `exclude: ['error']` keeps `console.error`.
  - The option is passed in `getBaseSWCOptions` (`next/dist/build/swc/options.js`), which both the server and client compiles use. Nothing limits it to the browser.
  - In that app's last local production build (`.next/`, `removeConsole: true` in `required-server-files.json`), the route handlers' `console.error("media image upload failed", …)`, `console.error("verify cron: app … failed", …)` and the `lost verification` line are gone from `.next/server`. One survived: `.catch((a) => console.error("verification-lost email failed", a))`, a call used as an expression. SWC removes `console.*` statements, not calls whose value is used.
  - So on that Worker, `src/` warnings such as `edge cache purge skipped` ([../03-caching/cache-invalidation-and-purge.md](../03-caching/cache-invalidation-and-purge.md)), `payload pg client error`, `revalidate skipped` and the `console.error` in `reportCaughtError` above are stripped. The PostHog capture still runs. `worker.ts` is bundled by wrangler, not Next, so its console calls survive, as do Next's, Payload's and other packages' own logs.
  - Not checked: the `.open-next` Worker bundle itself. OpenNext runs the same `next build`, so expect the same result.
