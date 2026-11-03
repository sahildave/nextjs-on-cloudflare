---
title: Custom domain and DNS cutover to Cloudflare Workers
description: "Put your own domain on a Worker: add the zone, copy every existing record, attach Worker custom domains, switch nameservers at your registrar, and keep a way back."
tags: [cookbook, cloudflare]
topics: [cloudflare, workers, dns, custom-domain, nameservers, cutover, workers-dev]
date: 2026-11-03
path: 04-domains/custom-domain-and-dns-cutover.md
---
# Custom domain and DNS cutover to Cloudflare Workers

You have a Worker running on `*.workers.dev` and a domain at any registrar, possibly still pointing at another host. This recipe moves the domain: it creates the Cloudflare zone, copies the records that matter, attaches the Worker as a custom domain (Cloudflare creates the DNS records for you), switches nameservers, and leaves the old host in place as the rollback. Done on three production domains on 3 Nov 2026. Moving from Vercel: the Vercel-specific records and steps are in [../migrate-from-vercel/README.md](../migrate-from-vercel/README.md).

## Prerequisites

- A deployed Worker you have tested on its `workers.dev` host: [../01-start/nextjs-on-workers-opennext.md](../01-start/nextjs-on-workers-opennext.md), [../01-start/deploy-from-github-actions.md](../01-start/deploy-from-github-actions.md).
- Access to the registrar (to change nameservers).
- Know which DNS records exist today: web, mail (MX, SPF, DKIM, DMARC), verification TXT, CAA. Mail records are their own recipe: [./email-dns-records.md](./email-dns-records.md).
- `www` goes to the apex through a Redirect Rule, as the base recipe sets out: [./www-redirect-rule.md](./www-redirect-rule.md). The `worker.ts` variant below is for a zone where you cannot add Redirect Rules.
- A staging host if you want a dress rehearsal first: [../06-operations/staging-and-preview-workers.md](../06-operations/staging-and-preview-workers.md).
- This recipe is for your own domain. Customer-owned domains on your Worker are a different feature: [./cloudflare-for-saas-custom-hostnames.md](./cloudflare-for-saas-custom-hostnames.md).
- Cron triggers and secrets already planned: [../05-app-features/cron-triggers.md](../05-app-features/cron-triggers.md), [../01-start/secrets-and-env.md](../01-start/secrets-and-env.md).

## Steps

1. A day ahead: lower the TTL on the current records at the old DNS host, so a rollback is quick.
2. Export the current records from the old DNS host (a BIND zone export, or its CLI's record list). Note also records the host serves by default but does not list as yours (see CAA under Gotchas).
3. Add the site in Cloudflare (Add a domain). Do not trust the quick scan: it imports proxied A records and can miss mail records. Delete what it imported and upload a zone file instead (DNS, Records, Import), with "Proxy imported DNS records" unticked. Leave the web records out; step 5 creates them.
4. Check the imported list against the export, record by record. Mail and verification records stay DNS-only (grey cloud).
5. In `wrangler.jsonc`, add the Worker custom domains and decide `workers_dev`. Do not deploy yet if the zone is still pending. See Code.
6. Remove any existing record for each name you are about to attach. A Worker custom domain cannot be created while a record exists for that name. (Skip if the zone has no web records yet.)
7. Switch the nameservers at the registrar to the two Cloudflare gave you. Resolvers keep the old ones for up to the registrar's NS TTL (3 hours for `.app`).
8. In the Cloudflare dashboard press "Check nameservers now" right after the switch, then wait for the zone to turn Active. On one domain that took about 40 minutes, and the certificate was ready about 2 minutes later.
9. Deploy the Worker (`wrangler deploy`). Wrangler creates the DNS records and certificates for each `custom_domain` route. Resolvers that already follow Cloudflare get no address for a name until the zone is active.
10. Verify (below). Then set `"workers_dev": false` for production and deploy again.
11. Turn on cron triggers now, not before, or the old host's crons and the Worker's run every job twice.
12. Keep the old project deployed but with git deploys off. Pause it later (a day or a week; see Gotchas).
13. Re-register anything that stores the host: webhook endpoints, OAuth redirect lists. With the same domain, usually nothing changes.

## Code

### `wrangler.jsonc` (production, after cutover)

```jsonc
{
  "name": "your-app",
  "main": "worker.ts",
  // Custom domains need the zone in this account; wrangler creates their DNS
  // records and certificates. www is not here: a Redirect Rule sends it to the apex.
  "routes": [{ "pattern": "your-domain.com", "custom_domain": true }],
  // Adding routes turns workers.dev off. Keep it true while you test the deploy
  // before cutover; set it to false once the domain is live. Workers Cache does
  // not key on host, so a workers.dev response (a noindexed one, for example)
  // can be served on the apex. *.workers.dev also sits outside the zone's WAF.
  "workers_dev": false,
  // Add at cutover, not before.
  "triggers": { "crons": ["0 4 * * *"] }
}
```

Before cutover, keep `"workers_dev": true` and `"crons": []`, then flip both in the cutover commit (crons on at cutover; `workers_dev` to `false` once the domain is live).

### Variant: `worker.ts` (www to apex, 308)

Only if `www` is a second Worker custom domain (add `{ "pattern": "www.your-domain.com", "custom_domain": true }` to `routes`). The Worker then starts for every `www` request.

```ts
export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);
    if (url.hostname === "www.your-domain.com") {
      url.hostname = "your-domain.com";
      return Response.redirect(url.toString(), 308);
    }
    return handler.fetch(request, env, ctx);
  },
} satisfies ExportedHandler;
```

### DNS records to carry over (one production list, as a template)

| Name | Type | Value | Note |
|---|---|---|---|
| `@` | CNAME (flattened) | old host's apex alias | removed when the Worker custom domain replaces it |
| `www` | CNAME | old host's CNAME | removed or proxied, see the www recipe |
| `send.email` | CNAME | mail provider | DNS-only for good |
| `resend._domainkey.email` | TXT | DKIM key, copied as is | DNS-only |
| `@` | TXT | search console verification | keep |
| old host's verification name | TXT | the old host's domain claim | keep for rollback; keeps the old host's claim valid |
| `@` | CAA | `0 issue "amazon.com"` and the others the old host serves | see Gotchas |

## Verify

1. Nameservers delegated:
   ```bash
   dig +short NS your-domain.com
   ```
   Expect the two Cloudflare names you were given.
2. Zone active, address resolves:
   ```bash
   dig +short your-domain.com @1.1.1.1
   ```
   Expect Cloudflare addresses. Empty means the zone is still pending.
3. The Worker answers on the real host:
   ```bash
   curl -sI https://your-domain.com/ | head -12
   ```
   Expect `HTTP/2 200`, `server: cloudflare`, and a `cf-ray` header. After the second request, `cf-cache-status: HIT` if Workers Cache is on ([../03-caching/workers-cache-edge-caching.md](../03-caching/workers-cache-edge-caching.md)).
4. www redirect:
   ```bash
   curl -sI "https://www.your-domain.com/path?q=1" | grep -iE "^HTTP|location"
   ```
   Expect `308` and `location: https://your-domain.com/path?q=1`.
5. workers.dev is off after the flip:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' https://your-app.<subdomain>.workers.dev/
   ```
   Expect `404` (the exact code may differ; it must not serve the site).
6. Placement, if pinned: `curl -sI https://your-domain.com/ | grep -i cf-placement` → `remote-FRA` (or your region).
7. Mail still works: send and receive a test message; check SPF/DKIM pass in the headers.
8. Watch the error tracker and `wrangler tail` for 48 hours.

## Gotchas

- Symptom: the dashboard quick scan imports the old host's IPs as proxied A records and misses the Resend (mail) records. Cause: the scan reads live resolvers, not the provider's zone. Fix: delete what it imported and upload a zone file (Import, proxy unticked). Mail records stay DNS-only.
- Symptom: Worker custom domain cannot be added. Cause: a record already exists for that name. Fix: delete the CNAME/A for that name first, then deploy.
- Symptom: `workers.dev` returns error 1042 or stops answering after adding `routes`. Cause: any `routes` entry turns workers.dev off. Fix: `"workers_dev": true` to test there before cutover.
- Symptom: the real domain serves a `noindex` response, or a preview-only response. Cause: workers.dev was left on; Workers Cache does not key on host, so its response got cached and served on the apex. A deploy clears it; the zone's "Purge Everything" does not. Fix: `"workers_dev": false` in production.
- Symptom: after switching nameservers some visitors get no address for minutes. Cause: a pending Cloudflare zone serves no Worker custom-domain records, and resolvers that already follow Cloudflare's nameservers get nothing until the zone is active (about 40 min on one domain, certificate about 2 min later). Fix: press "Check nameservers now" right after the switch, and do the cutover when a short gap is fine.
- Symptom: other resolvers still show the old host. Cause: NS TTL; 3 hours for `.app`. Fix: wait; do not pause the old host before then.
- Symptom: certificate issuance blocked. Cause: the old host serves default CAA records (for example `letsencrypt.org`, `pki.goog`, `sectigo.com`) that its record list does not show as yours, and an explicit `amazon.com` CAA blocks other CAs. Fix: copy the CAA set into the zone; Cloudflare adds its own CAs.
- Symptom: every cron job ran twice. Cause: triggers enabled on the Worker while the old host's crons still run. Fix: enable the Worker's crons at cutover, and turn the old crons off (git deploys off, project paused later).
- Symptom: users on slow ISP resolvers still reach the old host days later. Cause: ISP and machine DNS caches keep the old address for up to a day or two. Fix: do not pause the old project at cutover if anyone uses the site. A site with no visitors yet can pause it at once. Visitors on a stale cache who reach a paused project get a 503.
- Symptom: no Cloudflare plan features on the domain. Cause: plan benefits attach per zone and the zone must have its DNS on Cloudflare in the right account. Fix: add the zone to the account that holds the plan.
- Bot protection and WAF rules from the old host do not move. Rebuild them as Cloudflare WAF rules before cutover, and keep non-browser routes (`/api/*`, `.txt`, `.xml`) let through.
- Hosts that keep their names need no re-registration: Stripe live webhooks, Supabase redirect URLs and Apple return URLs stayed as they were on one move.
- Do the DNS move last. Test everything on `workers.dev` first, then on a staging custom domain, then move the apex.
- Rollback: for each moved name, detach the Worker custom domain, then re-add its CNAME to the old host from your export; for `www` delete the Redirect Rule and make the record DNS-only. Keep the old host's verification TXT records so its domain claim stays valid. Switching nameservers back is the full undo.
