---
title: www to apex redirect on Cloudflare Workers
description: "Redirect www.your-domain.com to the apex with a Cloudflare Redirect Rule, so the Worker serves only one hostname."
tags: [cookbook, cloudflare]
topics: [cloudflare, dns, redirect-rules, www]
date: 2026-11-03
path: 04-domains/www-redirect-rule.md
---
# www to apex redirect on Cloudflare Workers

A Worker custom domain serves exactly one hostname. If your app is on the apex and visitors also type `www.`, a Redirect Rule at the edge sends them over with a 308 and never starts the Worker. On Cloudflare you set this up yourself.

## Prerequisites

- The zone is on Cloudflare and the apex is a Worker custom domain: [./custom-domain-and-dns-cutover.md](./custom-domain-and-dns-cutover.md).
- A dashboard login with Rules edit on the zone.

## Steps

1. In DNS, add `www` as a record that points anywhere (a CNAME to the apex is fine) and set it **proxied** (orange cloud). Redirect Rules only run on proxied traffic. Do not add `www` as a Worker custom domain.
2. Rules > Redirect Rules > Create rule.
3. Match: hostname equals `www.your-domain.com`.
4. Action: dynamic redirect, status 308, preserve query string, URL `concat("https://your-domain.com", http.request.uri.path)`.
5. Deploy the rule.
6. Keep `www` out of `wrangler.jsonc` `routes`. Leave a comment saying why.

## Code

### Rule, as the dashboard's expression editor takes it

> **Unverified.** This expression is written from Cloudflare's Redirect Rules documentation. The production rule was recorded only in prose, so this exact text has not been run. Check it with the curl in Verify.

```
When:  (http.host eq "www.your-domain.com")
Then:  Dynamic redirect
       Expression: concat("https://your-domain.com", http.request.uri.path)
       Status: 308
       Preserve query string: on
```
The production rule in prose: host `www.your-domain.com`, 308 to `https://your-domain.com${path}`, keeping the query string.

### `wrangler.jsonc` (excerpt)
```jsonc
// www is not here: a Cloudflare Redirect Rule sends it to the apex (308).
"routes": [{ "pattern": "your-domain.com", "custom_domain": true }],
```

## Verify

```
curl -sI "https://www.your-domain.com/pricing?ref=x"
```
Expected: `HTTP/2 308` and `location: https://your-domain.com/pricing?ref=x`.

## Gotchas

- Symptom: `www` returns a Cloudflare error or times out. Cause: the record is DNS-only (grey cloud), so no rule runs. Fix: proxy it.
- Symptom: you cannot add `www` as a custom domain next to the apex redirect. Cause: you do not need to. The rule answers before any Worker. Fix: leave it out.
- Rollback: delete the rule and make the `www` record DNS-only again, pointing back at the old host.
