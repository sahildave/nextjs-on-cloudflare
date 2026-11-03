---
title: Customer custom domains with Cloudflare for SaaS on Workers
description: "Let your customers serve their own page from a domain they own (feedback.acme.com) on your Worker, with certificates issued by Cloudflare and no per-domain Worker config."
tags: [cookbook, cloudflare]
topics: [cloudflare, custom-hostnames, cloudflare-for-saas, multi-tenant, dns, tls, workers]
date: 2026-11-03
path: 04-domains/cloudflare-for-saas-custom-hostnames.md
---
# Customer custom domains with Cloudflare for SaaS on Workers

You need this when your product lets a customer put their content on their own hostname (a status page, a feedback board, a docs site). Cloudflare for SaaS adds each hostname to your zone as a "custom hostname", issues its certificate, and sends its traffic to your Worker. Your code adds, reads, re-checks and deletes hostnames over the REST API.

This recipe covers the zone setup, the API client, the status mapping for a settings pane and the tests. The Worker side (which tenant a Host header serves) is in [host-based-tenant-routing.md](./host-based-tenant-routing.md). Caching on such a Worker is in [workers-cache-host-gateway.md](../03-caching/workers-cache-host-gateway.md).

> **Not run live.** The pipeline had not added a real customer hostname when this was written. The API shapes come from Cloudflare's reference and the unit tests below. Treat the first live add as the real test.

This is not the recipe for putting your own domain on a Worker. For that, see [custom-domain-and-dns-cutover.md](./custom-domain-and-dns-cutover.md).

## Prerequisites

- A zone on Cloudflare (your product's domain, here `your-domain.com`) and a Worker deployed from it. Tested with Next 16.3.6, `@opennextjs/cloudflare` 1.20.15, wrangler 4.156.2.
- Cloudflare for SaaS enabled on the zone. A Free, Pro or Business zone includes 100 custom hostnames, then $0.10 each a month, up to 50,000.
- An API token scoped to that one zone with `Zone > SSL and Certificates > Edit`. Use one token per environment.
- A place to keep your own record of which tenant owns which host (for example a Supabase table). Cloudflare is not that record.

## Steps

1. Enable Cloudflare for SaaS: zone dashboard, SSL/TLS, Custom Hostnames.
2. Create the CNAME target. In DNS, add `AAAA domains 100::` with the proxy on (a proxied, originless record). Customers will CNAME to `domains.your-domain.com`.
3. On the Custom Hostnames page, set Fallback Origin to `domains.your-domain.com` and wait until it reads Active.
4. Route the zone to the Worker. Add the route `*/*` to the production Worker, at cutover and not before: it sends every proxied hostname on the zone to the Worker, your own host and every custom hostname. A more specific route (for example `staging.your-domain.com/*` to a staging Worker) wins over it. See Cloudflare's "Worker as origin" page.
5. Add the routes `*/.well-known/acme-challenge/*` and `*/.well-known/pki-validation/*` with Worker set to None, so the Worker never answers a certificate check. Cloudflare's DCV troubleshooting asks for this when a Worker is the fallback origin. It had not been seen to matter on a live certificate when this was written.
6. Put `CLOUDFLARE_API_TOKEN` and `CLOUDFLARE_ZONE_ID` in your secret store and push them to the Worker (see [secrets-and-env.md](../01-start/secrets-and-env.md)). A missing value must leave custom domains unavailable and the app booting.
7. Add the client (`src/lib/domains/cloudflare.ts`, `cloudflare-env.ts`, `types.ts`).
8. Order the attach flow: authorize the caller, check the plan, check your own record for a conflict, call Cloudflare, then write your record. If the write fails, delete the hostname from the zone again. Calling the provider before authorizing is the usual mistake.
9. Show the customer two DNS records and a status (see `state.ts`).

## Code

### `src/lib/domains/types.ts`

```ts
/** The hostname every customer CNAMEs to: the proxied record from step 2, also the zone's Fallback Origin. */
export const CUSTOM_DOMAIN_TARGET = "domains.your-domain.com";

export type TxtRecord = { name: string; value: string };

/** A custom hostname as the zone holds it (create, details and list all answer this). */
export type CustomHostname = {
  id: string;
  hostname: string;
  /** `pending` until ownership is proven, then `active`; also `moved`, `blocked`, ... */
  status: string;
  /** `initializing`, `pending_validation`, ..., `active`. Null before Cloudflare starts a certificate. */
  sslStatus: string | null;
  /** The `_cf-custom-hostname` TXT record that proves ownership without waiting on the CNAME. */
  ownership: TxtRecord | null;
  /** TXT records the certificate order still needs. Usually empty with HTTP validation. */
  certificateTxt: TxtRecord[];
};

export type DomainError =
  | { kind: "not_configured"; message: string }
  | { kind: "invalid_hostname"; message: string }
  | { kind: "domain_taken"; message: string }
  | { kind: "not_found"; message: string }
  | { kind: "rate_limited"; message: string; retryAfterSeconds: number | null }
  | { kind: "unauthorized"; message: string }
  | { kind: "network_error"; message: string }
  | { kind: "provider_error"; status: number; code: number | null; message: string };

export type DomainResult<T> = { ok: true; value: T } | { ok: false; error: DomainError };
```

### `src/lib/domains/cloudflare-env.ts`

```ts
export const CLOUDFLARE_TOKEN_ENV = "CLOUDFLARE_API_TOKEN";
export const CLOUDFLARE_ZONE_ID_ENV = "CLOUDFLARE_ZONE_ID";

export type CloudflareEnv = { token: string; zoneId: string };
export type CloudflareEnvResult = { ok: true; env: CloudflareEnv } | { ok: false; message: string };

/** A missing value is `{ ok: false }`, never a throw and never a crash at import time. */
export function getCloudflareEnv(): CloudflareEnvResult {
  if (typeof window !== "undefined") {
    return { ok: false, message: "The Cloudflare token is only available on the server." };
  }
  const token = process.env[CLOUDFLARE_TOKEN_ENV]?.trim();
  const zoneId = process.env[CLOUDFLARE_ZONE_ID_ENV]?.trim();
  if (!token || !zoneId) {
    const missing = !token ? CLOUDFLARE_TOKEN_ENV : CLOUDFLARE_ZONE_ID_ENV;
    return { ok: false, message: `${missing} is not set; custom domains are unavailable.` };
  }
  return { ok: true, env: { token, zoneId } };
}
```

### `src/lib/domains/cloudflare.ts`

Hand-written `fetch` against `https://developers.cloudflare.com/api/resources/custom_hostnames/`, no SDK. Server only. Errors are values; nothing throws across the seam.

```ts
import { getCloudflareEnv, type CloudflareEnv } from "./cloudflare-env";
import type { CustomHostname, DomainError, DomainResult, TxtRecord } from "./types";

const API = "https://api.cloudflare.com/client/v4";
const HOSTNAME = /^([a-z0-9]([a-z0-9-]{0,61}[a-z0-9])?\.)+[a-z]{2,63}$/;

/** "Duplicate custom hostname found" (409). */
const DUPLICATE = 1406;
/** Invalid, non-ASCII, reserved-TLD, unparseable or example.* hostnames (400). */
const INVALID_HOSTNAME_CODES = new Set([1407, 1408, 1409, 1410, 1411]);
/** "The custom hostname was not found." (404). */
const NOT_FOUND = 1436;

export function normalizeHostname(input: string): string | null {
  const host = input.trim().toLowerCase().replace(/\.$/, "");
  return host.length <= 253 && HOSTNAME.test(host) ? host : null;
}

export function attachDomain(domain: string): Promise<DomainResult<CustomHostname>> {
  return withEnv(domain, async (env, host) => {
    const created = await send(env, "POST", "/custom_hostnames", {
      hostname: host,
      ssl: { method: "http", type: "dv" },
    });
    return created.ok ? { ok: true, value: readHostname(created.value) } : created;
  });
}

export function getDomain(domain: string): Promise<DomainResult<CustomHostname>> {
  return withEnv(domain, async (env, host) => {
    const found = await find(env, host);
    return found.ok ? { ok: true, value: readHostname(found.value) } : found;
  });
}

/**
 * Validate again now instead of on Cloudflare's backoff schedule: a PATCH that
 * echoes the hostname's own ssl.method and ssl.type, which the docs name as the
 * configuration-neutral re-check. It also moves a `moved` hostname back to pending.
 */
export function recheckDomain(domain: string): Promise<DomainResult<CustomHostname>> {
  return withEnv(domain, async (env, host) => {
    const found = await find(env, host);
    if (!found.ok) return found;
    const ssl = record(found.value.ssl);
    const patched = await send(env, "PATCH", `/custom_hostnames/${String(found.value.id)}`, {
      ssl: {
        method: typeof ssl.method === "string" ? ssl.method : "http",
        type: typeof ssl.type === "string" ? ssl.type : "dv",
      },
    });
    return patched.ok ? { ok: true, value: readHostname(patched.value) } : patched;
  });
}

/** Idempotent: a hostname that is already gone is a success. Deleting it revokes its certificate. */
export function detachDomain(domain: string): Promise<DomainResult<null>> {
  return withEnv(domain, async (env, host) => {
    const found = await find(env, host);
    if (!found.ok) return found.error.kind === "not_found" ? { ok: true, value: null } : found;
    const removed = await send(env, "DELETE", `/custom_hostnames/${String(found.value.id)}`);
    if (!removed.ok && removed.error.kind !== "not_found") return removed;
    return { ok: true, value: null };
  });
}

async function withEnv<T>(
  domain: string,
  run: (env: CloudflareEnv, host: string) => Promise<DomainResult<T>>,
): Promise<DomainResult<T>> {
  const host = normalizeHostname(domain);
  if (!host) return fail({ kind: "invalid_hostname", message: `"${domain}" is not a valid hostname.` });
  const envResult = getCloudflareEnv();
  if (!envResult.ok) return fail({ kind: "not_configured", message: envResult.message });
  return run(envResult.env, host);
}

/** Cloudflare addresses a hostname by id and you store only the host, so list `?hostname=` first. */
async function find(env: CloudflareEnv, host: string): Promise<DomainResult<Record<string, unknown>>> {
  const listed = await send(env, "GET", `/custom_hostnames?hostname=${encodeURIComponent(host)}`);
  if (!listed.ok) return listed;
  const match = (Array.isArray(listed.value) ? listed.value : []).map(record).find((h) => h.hostname === host);
  return match ? { ok: true, value: match } : fail({ kind: "not_found", message: `${host} is not on the zone.` });
}

async function send(
  env: CloudflareEnv,
  method: "GET" | "POST" | "PATCH" | "DELETE",
  path: string,
  body?: unknown,
): Promise<DomainResult<unknown>> {
  let res: Response;
  try {
    res = await fetch(`${API}/zones/${env.zoneId}${path}`, {
      method,
      headers: {
        Authorization: `Bearer ${env.token}`,
        ...(body === undefined ? {} : { "Content-Type": "application/json" }),
      },
      body: body === undefined ? undefined : JSON.stringify(body),
      cache: "no-store",
    });
  } catch (err) {
    return fail({ kind: "network_error", message: err instanceof Error ? err.message : String(err) });
  }
  const json = record(await res.json().catch(() => null));
  if (!res.ok || json.success === false) return fail(toError(res, json));
  return { ok: true, value: json.result };
}

function fail(error: DomainError): { ok: false; error: DomainError } {
  return { ok: false, error };
}

function toError(res: Response, body: Record<string, unknown>): DomainError {
  const first = record(Array.isArray(body.errors) ? body.errors[0] : null);
  const code = typeof first.code === "number" ? first.code : null;
  const message = typeof first.message === "string" ? first.message : `Cloudflare answered ${res.status}.`;
  if (res.status === 409 || code === DUPLICATE) return { kind: "domain_taken", message };
  if (res.status === 429) {
    const retry = Number(res.headers.get("retry-after"));
    return { kind: "rate_limited", message, retryAfterSeconds: Number.isFinite(retry) && retry > 0 ? retry : null };
  }
  if (code !== null && INVALID_HOSTNAME_CODES.has(code)) return { kind: "invalid_hostname", message };
  if (res.status === 401 || res.status === 403) return { kind: "unauthorized", message };
  if (res.status === 404 || code === NOT_FOUND) return { kind: "not_found", message };
  return { kind: "provider_error", status: res.status, code, message };
}

/**
 * Not read, on purpose: ownership_verification_http, the http_url/http_body
 * validation records (with method "http" Cloudflare serves them itself once the
 * CNAME points here), verification_errors and ssl.validation_errors.
 */
function readHostname(body: unknown): CustomHostname {
  const b = record(body);
  const ssl = record(b.ssl);
  const ownership = record(b.ownership_verification);
  const records = Array.isArray(ssl.validation_records) ? ssl.validation_records.map(record) : [];
  return {
    id: String(b.id ?? ""),
    hostname: String(b.hostname ?? ""),
    status: String(b.status ?? "pending"),
    sslStatus: typeof ssl.status === "string" ? ssl.status : null,
    ownership:
      typeof ownership.name === "string" && typeof ownership.value === "string"
        ? { name: ownership.name, value: ownership.value }
        : null,
    certificateTxt: records
      .filter((r) => typeof r.txt_name === "string" && typeof r.txt_value === "string")
      .map((r): TxtRecord => ({ name: r.txt_name as string, value: r.txt_value as string })),
  };
}

function record(value: unknown): Record<string, unknown> {
  return typeof value === "object" && value !== null ? (value as Record<string, unknown>) : {};
}
```

### `src/components/settings/custom-domain/state.ts` (the readiness rule and the DNS rows)

Trimmed to the Cloudflare-specific part. The pane's other states (checking, removing, error copy) are your UI.

```ts
import { CUSTOM_DOMAIN_TARGET, type CustomHostname } from "@/lib/domains/types";

export type RecordRow = { type: "CNAME" | "TXT"; name: string; value: string };

const ACTIVE = new Set(["active", "active_redeploying"]);
const BLOCKED = new Set(["blocked", "pending_blocked", "test_blocked"]);

/** Cloudflare's own readiness rule: hostname active AND certificate active. */
export function isLive(h: Pick<CustomHostname, "status" | "sslStatus">): boolean {
  return ACTIVE.has(h.status) && h.sslStatus === "active";
}

export type DomainView =
  | { kind: "add-dns-record"; host: string; apex: boolean; records: RecordRow[] }
  | { kind: "issuing-certificate"; host: string; records: RecordRow[] }
  | { kind: "active"; host: string }
  | { kind: "error"; host: string; message: string };

export function statusView(host: string, h: CustomHostname): DomainView {
  if (BLOCKED.has(h.status)) return { kind: "error", host, message: "This domain can't be served." };
  if (isLive(h)) return { kind: "active", host };
  if (ACTIVE.has(h.status)) {
    return { kind: "issuing-certificate", host, records: h.certificateTxt.map((r) => txtRow(r, host)) };
  }
  return { kind: "add-dns-record", host, apex: looksLikeApex(host), records: dnsRecords(h, host) };
}

/** The CNAME to the target, then the ownership TXT. The CNAME alone validates once it resolves. */
export function dnsRecords(h: Pick<CustomHostname, "ownership">, host: string): RecordRow[] {
  const rows: RecordRow[] = [{ type: "CNAME", name: recordName(host, apexOf(host)), value: CUSTOM_DOMAIN_TARGET }];
  if (h.ownership) rows.push(txtRow(h.ownership, host));
  return rows;
}

function txtRow(r: { name: string; value: string }, host: string): RecordRow {
  return { type: "TXT", name: recordName(r.name, apexOf(host)), value: r.value };
}

/** The Name column is relative to the apex: `feedback`, not `feedback.acme.com` and not `feedback.`; `@` for the apex. */
export function recordName(fqdn: string, apexName: string): string {
  const name = fqdn.toLowerCase().replace(/\.$/, "");
  const apex = apexName.toLowerCase().replace(/\.$/, "");
  if (name === apex) return "@";
  return name.endsWith(`.${apex}`) ? name.slice(0, -(apex.length + 1)) : name;
}

/** Last two labels. A `co.uk`-style suffix is a known miss; use a public-suffix list if you need those. */
function apexOf(host: string): string {
  return host.trim().toLowerCase().replace(/\.$/, "").split(".").slice(-2).join(".");
}
export function looksLikeApex(host: string): boolean {
  return host.trim().replace(/\.$/, "").split(".").length === 2;
}
```

### Attach order, `src/features/domains/attach.ts` (skeleton)

```ts
export async function attachBoardDomain(input: { tenant: string; host: string }) {
  if (!(await mayChangeDomain(input.tenant))) return { ok: false, error: { kind: "refused" } };   // 1. authorize
  // 2. plan gate and limits, 3. your own conflict check: all before any Cloudflare call
  const host = normalizeHostname(input.host);
  if (host === null) return { ok: false, error: { kind: "invalid_hostname" } };
  if ((await tenantForHost(host)) !== null) return { ok: false, error: { kind: "domain_taken" } };

  const added = await addToZone(host);                                                               // 4. Cloudflare
  if (!added.ok) return added;
  try {
    await recordDomain({ host, tenant: input.tenant });                                              // 5. your record
  } catch (err) {
    await removeFromZone(host);   // the zone has it and you do not: take it back off
    throw err;
  }
  return { ok: true, domain: added.value };
}
```

### `src/features/domains/cloudflare-double.test-support.ts`

An in-memory zone as MSW handlers. Needed because no emulator of Cloudflare's custom-hostnames API existed (`emulate` has no Cloudflare service). It answers create, list by `?hostname=`, PATCH and delete; a new hostname is `pending` with `ssl.status: initializing`; a test moves it with `set`.

```ts
import { http, HttpResponse } from "msw";
import { server } from "@/test/server"; // your MSW server

const TOKEN = "cf_test_token";
export const ZONE = "zone_test";

type Hostname = { id: string; hostname: string; status: string; ssl: { method: string; type: string; status: string } };

export function startCloudflareDouble() {
  const zone = new Map<string, Hostname>();
  const calls: string[] = [];

  const body = (h: Hostname) => ({
    id: h.id,
    hostname: h.hostname,
    status: h.status,
    ownership_verification: { type: "txt", name: `_cf-custom-hostname.${h.hostname}`, value: `token-${h.id}` },
    ssl: { ...h.ssl, validation_records: [] },
  });
  const ok = (result: unknown, status = 200) => HttpResponse.json({ success: true, errors: [], result }, { status });
  const error = (status: number, code: number, message: string) =>
    HttpResponse.json({ success: false, errors: [{ code, message }], result: null }, { status });

  const admit = (request: Request, zoneId: string) => {
    const url = new URL(request.url);
    calls.push(`${request.method} ${url.pathname.replace(`/client/v4/zones/${zoneId}`, "")}${url.search}`);
    const authorized = request.headers.get("authorization") === `Bearer ${TOKEN}` && zoneId === ZONE;
    return authorized ? null : error(403, 10000, "Authentication error");
  };
  const byId = (id: string) => [...zone.values()].find((h) => h.id === id);

  const handlers = [
    http.post("https://api.cloudflare.com/client/v4/zones/:zone/custom_hostnames", async ({ request, params }) => {
      const refused = admit(request, String(params.zone));
      if (refused) return refused;
      const { hostname, ssl } = (await request.json()) as { hostname: string; ssl: { method: string; type: string } };
      if (zone.has(hostname)) return error(409, 1406, "Duplicate custom hostname found.");
      const created = { id: `ch-${hostname}`, hostname, status: "pending", ssl: { ...ssl, status: "initializing" } };
      zone.set(hostname, created);
      return ok(body(created), 201);
    }),
    http.get("https://api.cloudflare.com/client/v4/zones/:zone/custom_hostnames", ({ request, params }) => {
      const refused = admit(request, String(params.zone));
      if (refused) return refused;
      const wanted = new URL(request.url).searchParams.get("hostname");
      return ok([...zone.values()].filter((h) => wanted === null || h.hostname === wanted).map(body));
    }),
    http.patch("https://api.cloudflare.com/client/v4/zones/:zone/custom_hostnames/:id", ({ request, params }) => {
      const refused = admit(request, String(params.zone));
      if (refused) return refused;
      const found = byId(String(params.id));
      return found ? ok(body(found), 202) : error(404, 1436, "The custom hostname was not found.");
    }),
    http.delete("https://api.cloudflare.com/client/v4/zones/:zone/custom_hostnames/:id", ({ request, params }) => {
      const refused = admit(request, String(params.zone));
      if (refused) return refused;
      const found = byId(String(params.id));
      if (!found) return error(404, 1436, "The custom hostname was not found.");
      zone.delete(found.hostname);
      return ok({ id: found.id });
    }),
  ];

  return {
    calls,
    domains: () => [...zone.keys()],
    /** Move a hostname as Cloudflare would once the customer's DNS lands. */
    set(hostname: string, state: { status?: string; sslStatus?: string }) {
      const h = zone.get(hostname);
      if (!h) throw new Error(`cloudflare double: ${hostname} is not on the zone`);
      if (state.status) h.status = state.status;
      if (state.sslStatus) h.ssl.status = state.sslStatus;
    },
    reset() {
      zone.clear();
      calls.length = 0;
      process.env.CLOUDFLARE_API_TOKEN = TOKEN;
      process.env.CLOUDFLARE_ZONE_ID = ZONE;
      server.use(...handlers);
    },
  };
}
```

### `src/lib/domains/cloudflare.test.ts` (the shape of the unit tests)

Stub `fetch`, answer in the API's envelope, and assert the request.

```ts
import { afterEach, beforeEach, expect, it, vi } from "vitest";
import { attachDomain } from "./cloudflare";

const fetchMock = vi.fn<typeof fetch>();
const respond = (status: number, body: unknown) =>
  fetchMock.mockResolvedValueOnce(
    new Response(JSON.stringify(body), { status, headers: { "content-type": "application/json" } }),
  );

beforeEach(() => {
  vi.stubGlobal("fetch", fetchMock);
  vi.stubEnv("CLOUDFLARE_API_TOKEN", "tok_test");
  vi.stubEnv("CLOUDFLARE_ZONE_ID", "zone_test");
});
afterEach(() => {
  fetchMock.mockReset();
  vi.unstubAllGlobals();
  vi.unstubAllEnvs();
});

it("creates the normalized hostname with HTTP DV", async () => {
  respond(201, { success: true, errors: [], result: { id: "ch_1", hostname: "feedback.acme.com", status: "pending" } });
  await attachDomain(" Feedback.Acme.com ");
  const [url, init] = fetchMock.mock.calls[0];
  expect(String(url)).toBe("https://api.cloudflare.com/client/v4/zones/zone_test/custom_hostnames");
  expect(JSON.parse(String(init?.body))).toEqual({ hostname: "feedback.acme.com", ssl: { method: "http", type: "dv" } });
});

it("returns domain_taken for a duplicate (409, 1406)", async () => {
  respond(409, { success: false, errors: [{ code: 1406, message: "Duplicate custom hostname found." }], result: null });
  expect(await attachDomain("feedback.acme.com")).toMatchObject({ ok: false, error: { kind: "domain_taken" } });
});
```

## Verify

1. Add a hostname through your code, then read it back:

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/zones/<ZONE_ID>/custom_hostnames?hostname=feedback.acme.com" | jq '.result[0] | {status, ssl: .ssl.status}'
# before DNS:  { "status": "pending", "ssl": "initializing" }
# when ready:  { "status": "active",  "ssl": "active" }
```

2. After the customer's CNAME lands, `dig +short CNAME feedback.acme.com` returns `domains.your-domain.com.`.
3. `curl -sI https://feedback.acme.com/` returns 200 from your Worker with a valid certificate.
4. Delete the hostname in your UI. The list call above returns `result: []`.

The pipeline had not been run against a live customer hostname when this was written. The API shapes come from Cloudflare's reference and the tests above; the first live add is still to be confirmed.

## Gotchas

- Root domains (`acme.com`) → no address to give an A record → Cloudflare for SaaS has none without Apex Proxying, an Enterprise add-on → give a CNAME at `@` and tell the customer it works only where their DNS provider flattens CNAMEs (CNAME flattening, ALIAS, ANAME). A root domain on a provider without flattening, cannot be served.
- Staging and production share the zone → a hostname attached on one is a duplicate (409, code 1406) on the other → use separate test hostnames, or a separate zone for staging.
- Cloudflare addresses a hostname by id, your store keeps the host → read, re-check and delete need a lookup → list `?hostname=` first (one call per poll).
- Cloudflare's `verification_errors` and `ssl.validation_errors` are its and the CA's prose → not worth showing → a stalled domain gets generic tips. Say so in your docs.
- `*/*` route added too early → every hostname on the zone, including your production host, reaches the new Worker → add it at the production cutover, not during staging.
- Two custom-hostname caps to keep apart: Cloudflare's zone limit (100 included) and your own per-plan caps. Keep your per-plan caps (for example three per paid account) below the zone limit.
- Cloudflare's PATCH re-check is the "Check now" button. Without it, validation retries on Cloudflare's backoff schedule.
- Token scope: `SSL and Certificates` Edit, one zone. The token can add and remove hostnames, so keep it out of the browser (the env helper refuses to run when `window` exists).
