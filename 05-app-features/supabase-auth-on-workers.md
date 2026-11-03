---
title: Supabase Auth (SSR, OAuth) on Cloudflare Workers
description: "Supabase sign-in with @supabase/ssr on a Next.js Worker whose public pages sit in Workers Cache: proxy session refresh, a page-readable hint cookie, private auth paths, callbacks that survive any host."
tags: [cookbook, cloudflare]
topics: [cloudflare, workers, supabase, auth, oauth, cookies, nextjs-proxy, workers-cache, opennext]
date: 2026-11-03
path: 05-app-features/supabase-auth-on-workers.md
---
# Supabase Auth (SSR, OAuth) on Cloudflare Workers

Supabase Auth itself needs no change on Workers: `@supabase/ssr` and `supabase-js` talk to Supabase over `fetch`, so there is no socket or pool problem like Postgres has. What changes is the caching around it. Once Workers Cache serves your public pages without running the Worker, those pages cannot read the session cookie, and nothing that sets or depends on a cookie may be cached. This recipe is the cookie, proxy, callback and cache setup that keeps sign-in correct behind the edge cache.

## Prerequisites

- Next.js on Workers via OpenNext, with `proxy.ts` working: [../01-start/nextjs-on-workers-opennext.md](../01-start/nextjs-on-workers-opennext.md). Versions: Next 16.3.8, `@opennextjs/cloudflare` 1.20.15 (supports the Node `proxy.ts` since 1.20.x; the docs site lags the changelog), wrangler 4.156.x, `@supabase/ssr` 0.12.6, `@supabase/supabase-js` 2.115.0.
- Workers Cache wrapper that marks private paths `private, no-store`: [../03-caching/workers-cache-edge-caching.md](../03-caching/workers-cache-edge-caching.md). This recipe adds the auth rows to that list.
- `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY` and `NEXT_PUBLIC_APP_URL` present at build time. Next inlines `NEXT_PUBLIC_*` into the bundle during `next build`, so a Worker secret set later does not reach the browser: [../01-start/deploy-from-github-actions.md](../01-start/deploy-from-github-actions.md), [../01-start/secrets-and-env.md](../01-start/secrets-and-env.md).
- A Supabase project with an OAuth provider (Google, GitHub or Apple) configured.

## Steps

1. Pick the public origin once: `NEXT_PUBLIC_APP_URL=https://your-domain.com` (an origin, no path). Every redirect the auth code issues uses it, not the request's host.
2. In the Supabase dashboard (Auth, URL configuration): set Site URL to that origin and add `https://your-domain.com/auth/callback` to the redirect allow-list. If you test on a `*.workers.dev` or staging host, add `https://<that-host>/auth/callback` too. Provider-side callback URLs (Google, GitHub, Apple) point at Supabase, not at your Worker, so they do not change when you move hosting and keep the same domain.
3. Add `src/lib/supabase/cookies.ts` (below). It names the session cookie after your product and writes the page-readable hint.
4. Add `src/lib/supabase/server.ts` (a new client per request, bound to Next's cookie store) and the browser client of your choice.
5. Add `src/proxy.ts` and `src/lib/auth/proxy-session.ts`. The proxy refreshes the session and applies the sign-in gate. List only the routes that need a session in `config.matcher`.
6. Add `src/app/auth/callback/route.ts` (code exchange) and `src/app/auth/signout/route.ts` (POST). Both redirect to `NEXT_PUBLIC_APP_URL`.
7. Add `src/app/api/me/route.ts` returning the user with `Cache-Control: private, no-store`, plus a client-side provider that calls it. Public pages render signed out and fill in the user after load.
8. In `worker-cache.ts`, put `/auth`, `/api/`, and every gated page in the private prefix list. Keep the gate, the matcher and the private list in step with a test.
9. Deploy, sign in on the real domain, then run the Verify checks.

## Code

### `src/lib/supabase/cookies.ts`

The cookie name is per product. Supabase's default name is per project, so every product on one Supabase project would share a session on `localhost`. `withSignedInHint` adds the page-readable hint cookie to every session write.

A fuller file also has `stripProviderTokens`, which drops the GitHub `provider_token` from the session cookie before it is written. That is not a Workers concern, so it is left out here; its calls are marked where they go.

```ts
interface CookieWrite<O> {
  name: string;
  value: string;
  options: O;
}

/**
 * This product's Supabase cookie name, for all three clients. The default name
 * is per Supabase project, so every product on `localhost` that uses the same
 * Supabase project would share one session and one PKCE verifier.
 * `@supabase/ssr` maps `cookieOptions.name` to the storage key. It ends in
 * `-auth-token` so {@link AUTH_COOKIE} matches it.
 */
export const AUTH_COOKIE_NAME = "sb-your-app-auth-token";

/** Supabase's own auth cookies: `httpOnly`, unlike the library's default, since only the server reads them. */
export const SUPABASE_COOKIE_OPTIONS = {
  name: AUTH_COOKIE_NAME,
  httpOnly: true,
  sameSite: "lax",
  path: "/",
  secure: process.env.NODE_ENV === "production",
} as const;

/**
 * A non-secret, page-readable cookie that says only "a session exists". The
 * session cookie is httpOnly, so a static public page cannot see it; with this
 * one the browser asks `/api/me` for the user when there is one, and a
 * signed-out visitor costs one request per browser session. Its name must not match
 * {@link AUTH_COOKIE}, or it would be treated as a session cookie. Never
 * trusted for anything: the server reads the session itself.
 */
export const SIGNED_IN_HINT_COOKIE = "your-app-signed-in";

/**
 * The hint's values. `1`: a session exists. `0`: `/api/me` found none or a
 * sign-out cleared the session, so the
 * browser stops asking until the browser session ends or a sign-in writes `1`.
 * Absent: unknown (a session from before the hint, or a first visit), so the
 * browser asks `/api/me` once.
 */
export const SIGNED_IN_HINT_VALUES = { signedIn: "1", signedOut: "0" } as const;

/** `@supabase/ssr`'s own session cookie lifetime, which the `1` hint matches. */
export const SIGNED_IN_HINT_MAX_AGE = 400 * 24 * 60 * 60;

const { name: _sessionName, ...sessionCookieOptions } = SUPABASE_COOKIE_OPTIONS;

/** The hint's options when it is written outside a session write. */
export const SIGNED_IN_HINT_OPTIONS = {
  ...sessionCookieOptions,
  httpOnly: false,
} as const;

/**
 * One batch of Supabase cookie writes plus the matching hint write: `1` when
 * the batch writes a session, `0` for the browser session when it only clears
 * one (sign-out), so the next page skips `/api/me`. A batch that touches no
 * session cookie (the PKCE verifier) is returned unchanged.
 */
export function withSignedInHint<O extends object>(
  writes: readonly CookieWrite<O>[],
): CookieWrite<O>[] {
  const sessionWrites = writes.filter((w) => sessionCookieKey(w.name));
  const out = [...writes];
  if (sessionWrites.length === 0) return out;
  const live = sessionWrites.find((w) => w.value !== "");
  const options = (live ?? sessionWrites[0]).options;
  out.push(
    live
      ? {
          name: SIGNED_IN_HINT_COOKIE,
          value: "1",
          options: { ...options, httpOnly: false } as O,
        }
      : {
          name: SIGNED_IN_HINT_COOKIE,
          value: SIGNED_IN_HINT_VALUES.signedOut,
          options: { ...options, httpOnly: false, maxAge: undefined } as O,
        },
  );
  return out;
}

// `sb-<ref>-auth-token`, the PKCE `…-code-verifier`, and either one's `.N` chunks.
const AUTH_COOKIE = /^(sb-.+-auth-token)(-code-verifier)?(?:\.(\d+))?$/;

/** `sb-<ref>-auth-token` for the session cookie or one of its chunks — never the PKCE verifier. */
function sessionCookieKey(name: string): string | null {
  const match = AUTH_COOKIE.exec(name);
  return match && !match[2] ? match[1] : null;
}
```

### `src/lib/supabase/server.ts`

```ts
import { type CookieOptions, createServerClient } from "@supabase/ssr";
import type { SupabaseClient } from "@supabase/supabase-js";
import { cookies } from "next/headers";

import {
  SUPABASE_COOKIE_OPTIONS,
  withSignedInHint,
} from "@/lib/supabase/cookies";

/** The two methods of Next's cookie store the client uses. */
export type CookieStore = {
  getAll(): { name: string; value: string }[];
  set(name: string, value: string, options: CookieOptions): unknown;
};

/**
 * Supabase client for Server Components, route handlers and server functions.
 * A new client per request: never share one across requests (a module-level
 * client would keep one visitor's cookies for the next).
 */
export async function createClient() {
  return createClientWith(await cookies());
}

export function createClientWith(cookieStore: CookieStore): SupabaseClient {
  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL as string,
    process.env.NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY as string,
    {
      auth: { flowType: "pkce" },
      cookieOptions: SUPABASE_COOKIE_OPTIONS,
      cookies: {
        getAll() {
          return cookieStore.getAll();
        },
        setAll(cookiesToSet) {
          try {
            // Optionally wrap cookiesToSet in stripProviderTokens() here.
            for (const { name, value, options } of withSignedInHint(cookiesToSet)) {
              cookieStore.set(name, value, options);
            }
          } catch {
            // Server Components get a read-only cookie store. The proxy
            // refreshes the session on every matched navigation, so dropping
            // the write here is safe.
          }
        },
      },
    },
  );
}
```

### `src/lib/auth/proxy-session.ts`

Refreshes the session cookie, keeps the hint in step, and applies the sign-in gate before the page renders. Under Cache Components a page's `redirect()` streams a 200 first, so answering in the proxy gives a real 307. `routeDecision` is your own pure function (public vs gated paths); keep it in one file (`src/routes.ts`) with a table test.

```ts
import type { CookieMethodsServer } from "@supabase/ssr";
import { type NextRequest, NextResponse } from "next/server";

import {
  SIGNED_IN_HINT_COOKIE,
  SIGNED_IN_HINT_MAX_AGE,
  SIGNED_IN_HINT_OPTIONS,
  SIGNED_IN_HINT_VALUES,
  withSignedInHint,
} from "@/lib/supabase/cookies";
import { routeDecision } from "@/routes"; // (pathname, search, isLoggedIn) => {kind:"next"} | {kind:"redirect", to}

export type ProxyClientFactory = (cookies: {
  getAll: CookieMethodsServer["getAll"];
  setAll: NonNullable<CookieMethodsServer["setAll"]>;
}) => {
  auth: { getUser(): Promise<{ data: { user: unknown | null } }> };
} | null;

export async function runProxy(
  request: NextRequest,
  { createClient }: { createClient: ProxyClientFactory },
): Promise<NextResponse> {
  let response = NextResponse.next({ request });
  let refreshHeaders: Record<string, string> = {};

  const supabase = createClient({
    getAll() {
      return request.cookies.getAll();
    },
    setAll(cookiesToSet, headers) {
      const writes = withSignedInHint(cookiesToSet); // + stripProviderTokens, optionally
      for (const { name, value } of writes) {
        request.cookies.set(name, value);
      }
      response = NextResponse.next({ request });
      for (const { name, value, options } of writes) {
        response.cookies.set(name, value, options);
      }
      // Includes the no-store headers that keep a refreshed session out of a
      // CDN cache, where it could be served to a different user.
      for (const [key, headerValue] of Object.entries(headers)) {
        response.headers.set(key, headerValue);
      }
      refreshHeaders = headers;
    },
  });

  // Must run before the response is returned, or a refresh lands too late to
  // be written back to cookies.
  const user = supabase ? (await supabase.auth.getUser()).data.user : null;

  // Bring the page-readable hint in step with the session when no refresh
  // wrote it. The callback and sign-out handlers write it themselves.
  const { pathname } = request.nextUrl;
  const hasHint =
    request.cookies.get(SIGNED_IN_HINT_COOKIE)?.value ===
    SIGNED_IN_HINT_VALUES.signedIn;
  if (
    pathname !== "/auth/callback" &&
    pathname !== "/auth/signout" &&
    Boolean(user) !== hasHint &&
    !response.cookies.has(SIGNED_IN_HINT_COOKIE)
  ) {
    response.cookies.set(
      SIGNED_IN_HINT_COOKIE,
      user ? SIGNED_IN_HINT_VALUES.signedIn : "",
      { ...SIGNED_IN_HINT_OPTIONS, maxAge: user ? SIGNED_IN_HINT_MAX_AGE : 0 },
    );
  }

  // A redirect must carry the refresh's cookies and cache headers, or a
  // rotated session is lost on this request.
  const withRefresh = (other: NextResponse) => {
    for (const cookie of response.cookies.getAll()) other.cookies.set(cookie);
    for (const [key, headerValue] of Object.entries(refreshHeaders))
      other.headers.set(key, headerValue);
    return other;
  };

  const decision = routeDecision({
    pathname: request.nextUrl.pathname,
    search: request.nextUrl.search,
    isLoggedIn: Boolean(user),
  });
  if (decision.kind === "next") return response;

  const destination = new URL(decision.to, "http://x");
  const target = request.nextUrl.clone();
  target.pathname = destination.pathname;
  target.search = destination.search;
  return withRefresh(NextResponse.redirect(target, 307));
}
```

### `src/proxy.ts`

Next 16 renamed `middleware.ts` to `proxy.ts`. The matcher lists only the routes where a session matters. Public pages are not listed, so they are served with no Supabase call per request (and so Workers Cache can hold them). Next reads `matcher` as literals; it cannot be built from a shared constant.

```ts
import { createServerClient } from "@supabase/ssr";
import type { NextRequest } from "next/server";

import { runProxy } from "@/lib/auth/proxy-session";
import { SUPABASE_COOKIE_OPTIONS } from "@/lib/supabase/cookies";

export function proxy(request: NextRequest) {
  return runProxy(request, {
    createClient: (cookies) =>
      createServerClient(
        process.env.NEXT_PUBLIC_SUPABASE_URL as string,
        process.env.NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY as string,
        {
          auth: { flowType: "pkce" },
          cookieOptions: SUPABASE_COOKIE_OPTIONS,
          cookies,
        },
      ),
  });
}

export const config = {
  matcher: [
    "/dashboard/:path*",
    "/settings/:path*",
    "/submit/:path*",
    "/auth/:path*",
    "/api/upload-image",
  ],
};
```

### `src/app/api/me/route.ts` and `src/lib/auth/me.ts`

`/api/me` is the one request a static public page makes to learn who is signed in. It writes the hint either way, so a signed-out visitor asks once per browser session.

```ts
// src/lib/auth/me.ts
import { NextResponse } from "next/server";
import {
  SIGNED_IN_HINT_COOKIE,
  SIGNED_IN_HINT_MAX_AGE,
  SIGNED_IN_HINT_OPTIONS,
  SIGNED_IN_HINT_VALUES,
} from "@/lib/supabase/cookies";

export async function handleMe(deps: {
  currentUser: () => Promise<{ id: string; email?: string } | null>;
}): Promise<NextResponse> {
  const user = await deps.currentUser();
  const response = NextResponse.json(
    { user },
    { headers: { "Cache-Control": "private, no-store" } },
  );
  response.cookies.set(
    SIGNED_IN_HINT_COOKIE,
    user ? SIGNED_IN_HINT_VALUES.signedIn : SIGNED_IN_HINT_VALUES.signedOut,
    user
      ? { ...SIGNED_IN_HINT_OPTIONS, maxAge: SIGNED_IN_HINT_MAX_AGE }
      : SIGNED_IN_HINT_OPTIONS,
  );
  return response;
}

// src/app/api/me/route.ts
import { currentUser } from "@/lib/auth"; // getUser() via createClient(), cached per render
import { handleMe } from "@/lib/auth/me";

export function GET() {
  return handleMe({ currentUser });
}
```

### `src/lib/auth/client-user.ts`

```ts
import {
  SIGNED_IN_HINT_COOKIE,
  SIGNED_IN_HINT_VALUES,
} from "@/lib/supabase/cookies";

type MeResponse = { user: { id: string; email?: string } | null };

const ME_FAILED_AT = "your-app-me-failed-at";
/** After a failed `/api/me`, page loads in this window skip the request. */
const ME_RETRY_MS = 60_000;

export async function readCurrentUser({
  cookie,
  fetchMe = defaultFetchMe,
  storage = sessionStorageOrNull(),
  now = Date.now,
}: {
  cookie: string;
  fetchMe?: () => Promise<MeResponse>;
  storage?: Pick<Storage, "getItem" | "setItem"> | null;
  now?: () => number;
}): Promise<MeResponse["user"]> {
  const signedOut = cookie
    .split(";")
    .some(
      (part) =>
        part.trim() ===
        `${SIGNED_IN_HINT_COOKIE}=${SIGNED_IN_HINT_VALUES.signedOut}`,
    );
  if (signedOut) return null;
  const failedAt = storage?.getItem(ME_FAILED_AT);
  if (failedAt && now() - Number(failedAt) < ME_RETRY_MS) return null;
  try {
    return (await fetchMe()).user ?? null;
  } catch {
    storage?.setItem(ME_FAILED_AT, String(now()));
    return null;
  }
}

function sessionStorageOrNull() {
  try {
    return window.sessionStorage;
  } catch {
    return null;
  }
}

async function defaultFetchMe(): Promise<MeResponse> {
  const response = await fetch("/api/me", {
    credentials: "same-origin",
    cache: "no-store",
  });
  if (!response.ok) throw new Error(`/api/me answered ${response.status}`);
  return response.json();
}
```

Call `readCurrentUser({ cookie: document.cookie })` in a `useEffect` inside a client provider. The first render (the prerendered HTML) has no user and the avatar appears when `/api/me` answers. Protected layouts read the user on the server and pass it down.

### `src/app/auth/callback/route.ts` (the part that matters on Workers)

```ts
import { NextResponse } from "next/server";
import { createClient } from "@/lib/supabase/server";

const siteUrl = (path: string) => `${new URL(process.env.NEXT_PUBLIC_APP_URL as string).origin}${path}`;

export async function GET(request: Request) {
  const { searchParams } = new URL(request.url);
  const code = searchParams.get("code");
  if (!code) return NextResponse.redirect(siteUrl("/auth/login?error=unknown"));

  const supabase = await createClient();
  const { data, error } = await supabase.auth.exchangeCodeForSession(code);
  if (error || !data.user) {
    // A fixed key, not Supabase's message: the URL lands in history and logs.
    return NextResponse.redirect(siteUrl("/auth/login?error=exchange"));
  }
  // Redirect to the configured site URL, never the request origin: behind a
  // rewrite, a preview URL or the workers.dev host the request origin is not
  // the public one.
  return NextResponse.redirect(siteUrl(safeNext(searchParams.get("next"))));
}

// Guard `next` against open redirects: resolve it against a sentinel origin
// and compare origins and path.
const SENTINEL = "https://next.invalid";
function safeNext(next: string | null): string {
  const fallback = "/dashboard";
  if (!next || !next.startsWith("/")) return fallback;
  try {
    const url = new URL(next, SENTINEL);
    if (url.origin !== SENTINEL || url.pathname.startsWith("//") || url.pathname.startsWith("/\\"))
      return fallback;
    return `${url.pathname}${url.search}${url.hash}`;
  } catch {
    return fallback;
  }
}
```

A fuller callback also maps provider errors to fixed keys and creates a profile row after the exchange (signing the user out locally if that write fails): `src/lib/auth/callback.ts`.

### Sign-out

```ts
// src/app/auth/signout/route.ts
import { NextResponse } from "next/server";
import { createClient } from "@/lib/supabase/server";

export async function POST(request: Request) {
  // A form on another site can POST here. Judge by Sec-Fetch-Site, fall back to Origin.
  const site = request.headers.get("sec-fetch-site");
  const origin = request.headers.get("origin");
  const appOrigin = new URL(process.env.NEXT_PUBLIC_APP_URL as string).origin;
  const crossSite = site
    ? site !== "same-origin" && site !== "none"
    : origin !== null && origin !== appOrigin;
  if (crossSite) return new NextResponse(null, { status: 403 });

  const supabase = await createClient();
  // Scope "local": the default "global" ends the user's sessions on every
  // product that shares this Supabase user.
  await supabase.auth.signOut({ scope: "local" });
  return NextResponse.redirect(`${appOrigin}/`, 303);
}
```

### `src/lib/worker-cache.ts` (auth rows only)

The full wrapper is in [../03-caching/workers-cache-edge-caching.md](../03-caching/workers-cache-edge-caching.md). The auth-relevant parts are the private prefixes and the `Set-Cookie` rule.

```ts
export const PRIVATE_PREFIXES = [
  "/dashboard", "/settings", "/submit", // every gated page
  "/auth",                              // login, callback, signout
  "/api/",                              // /api/me and every route that reads the session
];

// In withSharedCache(): never keep a response that sets a cookie.
//   response.headers.has("set-cookie")  -> return it untouched, uncached
```

### Test: gated pages are matched and private

Keep the three lists in step with tests (`scripts/auth-proxy.test.ts`, `scripts/worker-cache.test.ts`): the first walks `src/app/**/(protected)` and fails when a page is missing from `config.matcher`; the second fails when a protected route from `src/routes.ts` is not in `PRIVATE_PREFIXES`. Write the same two tests for your layout. The matcher check uses Next's own tester:

```ts
import { unstable_doesMiddlewareMatch } from "next/experimental/testing/server";
import { config } from "../src/proxy";

// For each gated path:
const matches = unstable_doesMiddlewareMatch({ config, url: "http://localhost:3000/dashboard" });
// expect matches === true
```

## Verify

1. Unit and route tests (Node): `pnpm exec node --experimental-test-module-mocks --import tsx --test scripts/auth-*.test.ts` (or your equivalent). Expect the matcher test to list every gated page.
2. Sign-in flow on the real domain: open `https://your-domain.com/auth/login`, sign in with the provider. Expect a redirect to `https://your-domain.com/dashboard`, not to a `workers.dev` or preview host.
3. Cookies after sign-in (DevTools, Application, Cookies): `sb-your-app-auth-token` is `HttpOnly`, `Secure`, `SameSite=Lax`; `your-app-signed-in=1` is readable by the page.
4. Public page stays cached while signed in:
   ```bash
   curl -sI https://your-domain.com/ | grep -iE "cf-cache-status|set-cookie"
   ```
   Expect `cf-cache-status: HIT` (after the first request) and no `set-cookie`.
5. Auth and API paths are never cached:
   ```bash
   curl -sI https://your-domain.com/api/me | grep -iE "cache-control|cf-cache-status"
   ```
   Expect `cache-control: private, no-store` and no `HIT`.
6. Gate: `curl -sI https://your-domain.com/dashboard` signed out returns `307` with `location: /auth/login?callbackUrl=%2Fdashboard`.

## Gotchas

- Symptom: the signed-in avatar never appears on cached public pages, or every visitor sees the same avatar. Cause: a cached page cannot read the httpOnly session, and putting the user in the HTML would cache one person's identity for everyone. Fix: render signed out, and have the browser call `/api/me` (private, no-store). The hint cookie keeps signed-out visitors from calling it on every page.
- Symptom: a session refresh is served to the wrong person. Cause: a response that rotates the session cookie was cached. Fix: `withSharedCache` never keeps a response with `Set-Cookie`, the proxy copies Supabase's no-store headers onto the response, and `/auth`, `/api/` and gated pages are private paths. Put a new gated route in the matcher, the route table and the private list (three places); the two tests above catch a miss.
- Symptom: a gated page loads for signed-out visitors, or its session never refreshes. Cause: the route is not in `config.matcher`. Fix: add it. Next reads the matcher as literals, so no shared constant.
- Symptom: after sign-in the browser lands on the `workers.dev` or preview host and the session cookie is for the wrong host. Cause: the callback redirected to `request.url`'s origin. Fix: redirect to the configured `NEXT_PUBLIC_APP_URL`.
- Symptom: `redirect_to` rejected, or the provider returns to Supabase's Site URL instead of your callback. Cause: the host is not in Supabase's redirect allow-list. Fix: add each host you sign in on (apex, staging, a preview host if you test there). When the domain keeps its name through a hosting move, nothing changes.
- Symptom: `NEXT_PUBLIC_SUPABASE_URL is not set` at runtime, or the browser bundle has `undefined`. Cause: the value was only a Worker secret. `NEXT_PUBLIC_*` is inlined at `next build`. Fix: export them into the build step's environment (and as Worker secrets only for server-side reads).
- Symptom: `supabase.auth.getUser()` in the proxy adds latency to every request. Cause: it is a network call to Supabase. Fix: that is why the matcher is narrow. Public pages never run it.
- Symptom: signing out of one product signs the user out of others. Cause: Supabase's default sign-out scope is global. Fix: `signOut({ scope: "local" })`, and a per-product cookie name.
- Symptom: after a deploy, a tab open across it reloads once on navigation. Cause: build mismatch, normal. Not an auth bug.
- Passwords: this recipe uses OAuth only. Email and password sign-in is hashed by Supabase on its side, not on the Worker, so it hits no PBKDF2 cap. (Payload's own admin login does: [./payload-cms-on-workers.md](./payload-cms-on-workers.md).)
- Not measured: the cost of `getUser()` in the proxy on a gated page from a distant visitor, and cookie behaviour on `*.workers.dev` versus the apex (they are different hosts, so they never share a session).
