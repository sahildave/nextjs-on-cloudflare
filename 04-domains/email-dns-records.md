---
title: Keeping email DNS records when moving a zone to Cloudflare
description: "Move a domain's DNS to Cloudflare without breaking Resend sending, Search Console, CAA or the records you need for a rollback."
tags: [cookbook, cloudflare]
topics: [cloudflare, dns, email, resend, caa, zone-import]
date: 2026-11-03
path: 04-domains/email-dns-records.md
---
# Keeping email DNS records when moving a zone to Cloudflare

You need this when the nameservers move from another DNS host to Cloudflare and the domain sends mail through Resend. Cloudflare's automatic scan imports the wrong things and misses the email records. A hand-checked zone file avoids that.

## Prerequisites

- Registrar access to change nameservers.
- The old host's full record list (a zone export or its CLI), plus Resend's domain page for its three CNAMEs and DKIM TXT. Coming from Vercel: [../migrate-from-vercel/README.md](../migrate-from-vercel/README.md).
- Apex and subdomain cutover steps: [./custom-domain-and-dns-cutover.md](./custom-domain-and-dns-cutover.md).

## Steps

1. Add the zone in Cloudflare. Discard the records its quick scan imports: it brings the old host's IPs in as proxied A records and misses the Resend records.
2. Build a zone file from the old host's list (see Code). Include the email records, the Search Console TXT, CAA and the old host's verification TXT.
3. DNS > Records > Import. Untick "Proxy imported DNS records". Everything stays DNS-only until its own cutover step.
4. Diff the Cloudflare zone against the old list record by record before you touch nameservers.
5. Switch the nameservers at the registrar. Resolvers may keep the old ones for the NS TTL (`.app` is 3 hours).
6. Move names one at a time to Worker custom domains. Delete the placeholder CNAME for a name first; a custom domain cannot be added while a record exists.
7. Keep the old host's verification TXT records until you are sure you will not roll back.

## Code

### Zone file (`your-domain.com.zone`), email and verification records

> **Unverified.** Assembled from one production migration table, not copied from a file that was imported. Diff it against your old host's records before you upload it.
>
> The Resend CNAME targets below (`send.forge.rmta.net`, `rsend-euw1.forge.rmta.net`, `links2.resend-dns.com`) and the CAA set are one account's values, tied to its sending region. Take yours from Resend's domain page and your old host's records.

```
; Resend sending. Keep DNS-only (grey cloud) permanently.
send.email.your-domain.com.         300 IN CNAME send.forge.rmta.net.
rsend.email.your-domain.com.        300 IN CNAME rsend-euw1.forge.rmta.net.
links.email.your-domain.com.        300 IN CNAME links2.resend-dns.com.
resend._domainkey.email.your-domain.com. 300 IN TXT "<DKIM_PUBLIC_KEY_COPIED_AS_IS>"

; Search Console
your-domain.com.                    300 IN TXT "google-site-verification=<TOKEN>"

; Old host's domain claims, kept for rollback (name and value from the old host)
<old-host-verify-name>.your-domain.com. 300 IN TXT "<old host's verification value>"

; CAA: copy the old host's defaults, or an explicit single CA blocks the rest
your-domain.com.                    300 IN CAA 0 issue "amazon.com"
your-domain.com.                    300 IN CAA 0 issue "letsencrypt.org"
your-domain.com.                    300 IN CAA 0 issue "pki.goog"
your-domain.com.                    300 IN CAA 0 issue "sectigo.com"
```
The migration table also listed `staging` and `www` CNAMEs to the old host and an apex flattened CNAME, which it then replaced with Worker custom domains and a [Redirect Rule](./www-redirect-rule.md). The CAA lines for `letsencrypt.org`, `pki.goog` and `sectigo.com` were the old host's defaults, which its record list did not show as your own. The source documented the records, not a zone file; this file is assembled from that table.

## Verify

```
dig +short NS your-domain.com                      # Cloudflare nameservers
dig +short CNAME send.email.your-domain.com        # send.forge.rmta.net.
dig +short CAA your-domain.com                     # the CAs you kept
```
Then confirm the domain shows "Verified" in Resend and send a test message.

## Gotchas

- Symptom: Resend marks the domain unverified after the move. Cause: the dashboard scan did not import the three CNAMEs and the DKIM TXT. Fix: import a zone file with them.
- Symptom: Resend records are proxied and mail fails. Cause: orange cloud on a mail-related CNAME. Fix: DNS-only, permanently.
- Symptom: certificate issuance fails for an unlisted CA. Cause: you copied only one CAA record (`amazon.com`), which blocks every other CA. Fix: copy all of the old host's CAA records. Cloudflare adds its own CAs.
- Symptom: the old host lists no MX or SPF record and you add some. Cause: Resend's `send.email` CNAME already covers both for the sending subdomain. Fix: do not add duplicates.
- Symptom: rollback impossible. Cause: you deleted the old host's verification TXT claims. Fix: keep them.
- The dashboard import wants "Proxy imported DNS records" unticked. Leaving it on proxies the old host's IPs.
