---
title: Give workerd's outbound fetch a User-Agent
description: "Stop APIs that reject requests with no User-Agent (GitHub's returns 403) from failing only on Workers, by defaulting the header on every outbound fetch."
tags: [cookbook, cloudflare]
topics: [cloudflare, fetch, user-agent, github-api, workerd, nodejs-compat]
date: 2026-11-03
path: 05-app-features/outbound-fetch-user-agent.md
---
# Give workerd's outbound fetch a User-Agent

Node's `fetch` sends a `User-Agent`; workerd's does not. GitHub's REST and GraphQL APIs answer 403 to a request without one. The same code works under `next dev` and fails on the Worker. In one app every page that checked whether a repository was public showed it as private, because that check got a 403. This recipe wraps `globalThis.fetch` once in `worker.ts` so any request without a `User-Agent` gets yours.

## Prerequisites

- A `worker.ts` entry that you own (see [nextjs-on-workers-opennext.md](../01-start/nextjs-on-workers-opennext.md)).
- wrangler 4.156.x. `global_fetch_strictly_public` in the compatibility flags does not matter for this recipe.

## Steps

1. At the top of `worker.ts`, before the handler is used, save the original `fetch` and replace `globalThis.fetch` with a function that adds the header when missing.
2. Use a stable product name for the value (`your-app`). GitHub only requires that one exists.
3. Rebuild and deploy. No change to calling code.

## Code

### `worker.ts` (top of file)

```ts
// Node's fetch sends a User-Agent and workerd's does not. GitHub's API refuses a
// request without one (403), and the app calls it from many places, so every
// outbound request that sets none gets the app's here.
const workerdFetch = globalThis.fetch;
globalThis.fetch = (input: RequestInfo | URL, init?: RequestInit) => {
  const request = new Request(input, init);
  if (request.headers.has("user-agent")) return workerdFetch(request);
  const headers = new Headers(request.headers);
  headers.set("user-agent", "your-app");
  return workerdFetch(new Request(request, { headers }));
};
```

## Verify

1. Before the fix, from the deployed Worker, call a route that hits `https://api.github.com/repos/<owner>/<repo>`: it answers 403.
2. After: the same route returns 200.
This was checked on a `*.workers.dev` test Worker, not locally.

## Gotchas

- Works under `next dev`, 403 on the Worker → Node adds a `User-Agent` and workerd does not → the wrapper.
- A caller that passes a `Request` object with its own headers keeps them; the wrapper only fills a missing `user-agent`.
- Untested: an SDK that captures `fetch` at import time, before the wrapper runs, would keep the original. If you use one, pass `fetch` in explicitly.
