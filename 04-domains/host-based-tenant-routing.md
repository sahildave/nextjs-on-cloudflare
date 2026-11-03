---
title: Host-based tenant routing in Next on Workers
description: "Serve a different tenant's pages per Host header (app.acme.com), with the host lookup cached per host and expired the moment a domain is attached or removed."
tags: [cookbook, cloudflare]
topics: [cloudflare, multi-tenant, custom-hostnames, nextjs, proxy, cache-tags]
date: 2026-11-03
path: 04-domains/host-based-tenant-routing.md
---
# Host-based tenant routing in Next on Workers

You need this once customer hostnames reach your Worker (see [cloudflare-for-saas-custom-hostnames.md](./cloudflare-for-saas-custom-hostnames.md)) and the app must pick the right tenant from the `Host` header. The pattern: `proxy.ts` rewrites every customer-host request into a `/host/*` route tree by string work alone, and a `"use cache"` function maps host to tenant, tagged per host so attach and detach take effect on the next request.

## Prerequisites

- Next 16 (the file is `src/proxy.ts`; it was `middleware.ts` before 16), Cache Components with `"use cache"`.
- OpenNext on Workers with the tag cache set up. See [cache-components-ppr-on-workers.md](../03-caching/cache-components-ppr-on-workers.md).
- A table mapping host to tenant (for example `custom_domains` in Supabase).
- Caching public pages on such a Worker needs a host check first: [workers-cache-host-gateway.md](../03-caching/workers-cache-host-gateway.md).

## Steps

1. Add `requestHost` (below). The `Host` header is attacker-supplied and becomes a cache key, so bound it to DNS characters and 251 characters (a cache tag is capped at 256 and is `host:` plus the host).
2. In `src/proxy.ts`, decide app host or customer host. On the app host do nothing (or your own work). On a customer host: let `/_next/*` and a short pass-through list through, answer 404 for `/api/*`, and rewrite everything else `P` to `/host{P}`. A path with no page under `/host` (the landing page, pricing) is then a 404 on the customer's domain.
3. Keep the proxy free of I/O. It compiles into its own module graph and cannot read your store anyway. Hold this with a test that checks the proxy's import list.
4. Add the `host` route tree (`src/app/host/...`) that reuses your tenant layout and pages. The segment is `host`, not `_host`: Next treats `_`-prefixed folders as private, so they are not routes.
5. Split the lookup in two, because a `"use cache"` scope must not read request data: `tenantForRequestHost` reads `headers()` outside the cache and passes the host in as an argument (which makes it part of the cache key); `resolveTenantForHost` carries `"use cache"` and `cacheTag(hostTag(host))`.
6. Cache the unknown host too (the null answer carries the tag). On attach and detach call `revalidateTag(hostTag(host), { expire: 0 })`, so a newly attached host serves immediately and a removed one stops immediately.
7. Narrow the proxy `config.matcher` so static files do not cost a run: it is a billed invocation per request. Do not skip by file extension alone when your tenant paths can contain dots (see Gotchas).
8. Treat `*.workers.dev` and loopback as app hosts. No customer can attach `*.workers.dev`, so any request there is one of your own deployments.

## Code

### `src/lib/domains/host.ts`

```ts
/** Lower-cased, trailing dot of a fully qualified name stripped. The one place a host is normalised. */
export function normaliseHost(host: string): string {
  return host.trim().toLowerCase().replace(/\.$/, "");
}

/** A cache tag is capped at 256 characters and is `host:` + the host, so 251. */
export const MAX_REQUEST_HOST_LENGTH = 251;

/**
 * A `Host` header as routing reads it, or null: port dropped, normalised, bounded to
 * letters, digits, dots and hyphens. The header is attacker-supplied and becomes a
 * cache key. An IPv6 literal is refused too.
 * Pure string work: proxy's module graph must stay free of I/O.
 */
export function requestHost(header: string | null): string | null {
  if (header === null) return null;
  const host = normaliseHost(header.replace(/:\d*$/, ""));
  if (host === "" || host.length > MAX_REQUEST_HOST_LENGTH) return null;
  return /^[a-z0-9](?:[a-z0-9.-]*[a-z0-9])?$/.test(host) ? host : null;
}
```

### `src/lib/domains/tags.ts`

```ts
export function hostTag(host: string): string {
  return `host:${host}`;
}

/** Lets a page on your own domain emit rel=canonical to the tenant's custom host, and move back on detach. */
export function tenantDomainTag(tenant: string): string {
  return `tenant-domain:${tenant}`.toLowerCase();
}
```

### `src/lib/domains/resolve-host.ts`

```ts
import { cacheTag } from "next/cache";
import { headers } from "next/headers";
import { requestHost } from "./host";
import { tenantForHost, type TenantRef } from "./store";
import { hostTag } from "./tags";

/** The tenant this request's host serves, or null. Reads `headers()`; caches nothing itself. */
export async function tenantForRequestHost(): Promise<TenantRef | null> {
  const host = requestHost((await headers()).get("host"));
  return host === null ? null : resolveTenantForHost(host);
}

/** The cached lookup, keyed on `host` (an argument is part of the cache key). */
export async function resolveTenantForHost(host: string): Promise<TenantRef | null> {
  "use cache";
  cacheTag(hostTag(host));
  return tenantForHost(host);
}
```

### `src/lib/domains/store.ts` (the expiry, excerpt)

```ts
import { revalidateTag } from "next/cache";

export async function attachDomain(input: { host: string; tenant: string }) {
  const host = normaliseHost(input.host);
  if (host === "") throw new Error("attachDomain: empty host");
  const domain = await store.attach({ host, tenant: input.tenant });
  revalidateTag(hostTag(host), { expire: 0 });
  revalidateTag(tenantDomainTag(input.tenant), { expire: 0 });
  return domain;
}
// detachDomain does the same two revalidateTag calls after store.detach.
```

### `src/proxy.ts` (host routing half)

```ts
import { NextResponse, type NextRequest } from "next/server";
import { requestHost } from "@/lib/domains/host";

const PASS_THROUGH = new Set(["/favicon.ico", "/icon.svg", "/api/auth/redeem"]);

export const config = {
  matcher: [
    // Skip static files. A dotted tenant path (/t/acme/x.png, /t/next.js/about) must still match.
    "/((?!_next/static/|_next/image|[^/]+\\.(?:png|jpg|jpeg|gif|webp|avif|svg|ico|woff|woff2|ttf|otf)$|(?:content|fonts|mock|schema)/.*\\.(?:png|jpg|jpeg|gif|webp|avif|svg|ico|woff|woff2|ttf|otf)$).*)",
  ],
};

export function proxy(request: NextRequest): NextResponse {
  const { pathname } = request.nextUrl;
  // An analytics reverse proxy must pass on every host, or it is rewritten to /host/ingest and the event is lost.
  if (pathname === "/ingest" || pathname.startsWith("/ingest/")) return NextResponse.next();

  const host = requestHost(request.headers.get("host"));
  if (host === null || isAppHost(host)) return NextResponse.next(); // your app-host logic goes here

  if (pathname.startsWith("/_next/") || PASS_THROUGH.has(pathname)) return NextResponse.next();
  if (pathname === "/api" || pathname.startsWith("/api/")) return new NextResponse(null, { status: 404 });

  const url = request.nextUrl.clone();
  url.pathname = pathname === "/" ? "/host" : `/host${pathname}`;
  return NextResponse.rewrite(url);
}

function isAppHost(host: string): boolean {
  if (host === "localhost" || host === "127.0.0.1" || host.endsWith(".workers.dev")) return true;
  const site = process.env.NEXT_PUBLIC_SITE_URL;
  return site !== undefined && site !== "" && requestHost(site.replace(/^[a-z]+:\/\//i, "").replace(/\/.*$/, "")) === host;
}
```

### `src/app/host/(tenant)/layout.tsx`

```tsx
import type { ReactNode } from "react";
import TenantLayout from "@/app/t/[tenant]/layout";
import { hostParams } from "@/lib/domains/host-params";

// Imports the tenant layout instead of copying it, and hands it params resolved from the Host header.
export default function HostLayout({ children }: { children: ReactNode }) {
  return <TenantLayout params={hostParams()}>{children}</TenantLayout>;
}
```

### `src/lib/domains/host-params.ts`

> **Unverified.** Written for this cookbook from the production file's contract (resolve the host, 404 when no tenant has it, return the layout's params); the file itself was not copied.

```ts
import { notFound } from "next/navigation";
import { tenantForRequestHost } from "./resolve-host";

/** The params `/t/[tenant]` expects, resolved from the Host header. A host with no tenant is a 404. */
export async function hostParams(): Promise<{ tenant: string }> {
  const tenant = await tenantForRequestHost();
  if (tenant === null) notFound();
  return { tenant: tenant.slug };
}
```

Next 16 passes layout `params` as a promise, so the layout hands `hostParams()` over without awaiting it. Every page under `src/app/host/(tenant)/` does the same.

### `src/lib/domains/host.test.ts`

```ts
import { describe, expect, it } from "vitest";
import { normaliseHost } from "./host";

describe("normaliseHost", () => {
  it.each([
    ["app.acme.com", "app.acme.com"],
    ["APP.ACME.COM.", "app.acme.com"],
    [" App.Acme.Com. ", "app.acme.com"],
  ])("%j → %j", (input, expected) => {
    expect(normaliseHost(input)).toBe(expected);
  });
});
```

## Verify

With a custom hostname active and a test tenant attached:

```bash
curl -s https://app.acme.com/ -o /dev/null -w "%{http_code}\n"        # 200, the tenant page
curl -s https://app.acme.com/pricing -o /dev/null -w "%{http_code}\n" # 404, your marketing page is not served there
curl -s https://app.acme.com/api/anything -o /dev/null -w "%{http_code}\n" # 404
```

1. Attach a new host and request it within a second. It serves the tenant, not a cached null.
2. Detach it and request again. It returns 404 on the next request.

## Gotchas

- Skipping static files by extension in the matcher → a tenant path that ends in a file extension (`/t/acme/x.png`, a tenant named `next.js`) skips the proxy → on a customer host it renders another tenant's page under the customer's domain → skip only root-level files and named `public/` directories; do not skip `.js`, `.css`, `.txt`, `.xml`, `.md`.
- A new `public/` directory → its files cost a proxy run and 404 on a customer host → add the directory to the matcher.
- Reverse proxy paths (analytics `/ingest`) rewritten to `/host/ingest/...` → events lost → pass them through first, on every host.
- `"use cache"` reading `headers()` → build error → read the header outside, pass the host in as an argument.
- Only Cloudflare-routed hosts reach the Worker: the zone's own hosts and the custom hostnames it holds. The exposure of an attacker-chosen `Host` is therefore narrow, but still bound it before it becomes a cache key.
- A customer host gets no markdown twins, robots or sitemap here; if you want them per tenant, route them under `/host` too.
- Public-page caching in front of the Worker is keyed by path, not host. A customer host's `/` must never reach it. Read [workers-cache-host-gateway.md](../03-caching/workers-cache-host-gateway.md) before turning Workers Cache on.
