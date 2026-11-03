<p align="center"><img src="banner.png" alt="Next.js on Cloudflare Workers"></p>

# Next.js on Cloudflare Workers: a cookbook for agents

**26 recipes for running a Next.js 16 app on Cloudflare Workers** with OpenNext (`@opennextjs/cloudflare`), plus a guide for moving off Vercel.

Each recipe covers one feature, stands on its own, and gives an agent the code, the checks and the failures to expect.

### What it covers

* **Setup:** `wrangler.jsonc`, `open-next.config.ts` and a custom `worker.ts`; plan limits; deploys from GitHub Actions; secrets and build-time env.
* **Data:** Postgres through Hyperdrive with `pg` or `postgres.js`, and R2 writes for the page cache.
* **Caching:** Workers Cache in front of the app, tag invalidation and purge, and patches for Cache Components (PPR) and `"use cache"` hangs on Workers.
* **Domains:** DNS cutover, the `www` redirect, email records, customers' own domains through Cloudflare for SaaS, and routing by host.
* **App features:** images without `sharp`, Payload CMS, Supabase Auth, cron triggers, and outbound `fetch`.
* **Operations:** logs, a separate preview Worker for branches, and an incident playbook.

### Where the recipes come from

Three Next.js apps that moved to Workers and run there in production:

* <img src="https://raw.githubusercontent.com/sahildave/coduo-skills/main/assets/apps/macappspace.png" width="16" alt=""> [macapp.space](https://macapp.space): a directory of Mac apps. Payload CMS, Supabase Postgres and Auth, Stripe for sponsor slots.
* <img src="https://raw.githubusercontent.com/sahildave/coduo-skills/main/assets/apps/shareling.png" width="16" alt=""> [shareling.app](https://shareling.app): the web companion to a Mac app. Supabase Auth, Postgres, Stripe.
* <img src="https://raw.githubusercontent.com/sahildave/coduo-skills/main/assets/apps/gitback.png" width="16" alt=""> [gitback.us](https://gitback.us): GitHub-native feedback boards for many teams, each of which can serve its board on its own domain.

Every quirk in a recipe was hit on one of these apps, and the recipe gives the symptom, the cause and the fix.

> [!NOTE]
> Code we did not run in the exact form shown is marked **Unverified**. Test it before you rely on it.

### Versions

| Package | Version |
|---|---|
| Next.js | 16.3.x |
| `@opennextjs/cloudflare` | 1.20.15 |
| `wrangler` | 4.156.2 |
| Node.js | 24 |

Patches target these exact versions. Regenerate them against yours.

Last updated: 2026-11-03

## Structure

| Folder | What is in it |
|---|---|
| `01-start/` | OpenNext setup, plans and limits, GitHub Actions deploys, secrets and env |
| `02-data/` | Hyperdrive with Supabase Postgres, postgres.js, R2 cache writes |
| `03-caching/` | Workers Cache, invalidation and purge, Cache Components fixes |
| `04-domains/` | DNS cutover, www redirect, email records, customer domains, tenant routing |
| `05-app-features/` | Images, Payload CMS, Supabase Auth, cron triggers, outbound fetch |
| `06-operations/` | Logs and error tracking, staging Workers, incident playbook |
| `migrate-from-vercel/` | The Vercel-specific steps, plus a case study of one move |

Each recipe has Prerequisites, Steps, Code, Verify and Gotchas.

## Use it with an agent

**As a skill.** Paste this into your agent:

```
Install the skill from https://github.com/sahildave/nextjs-on-cloudflare by cloning it into your skills folder as nextjs-on-cloudflare (for Claude Code: ~/.claude/skills/nextjs-on-cloudflare). Then use it to deploy this Next.js app to Cloudflare Workers: follow its start-here order, load one recipe at a time, and run each recipe's Verify step before moving on.
```

**As a reference.** Without installing anything:

```
Read https://github.com/sahildave/nextjs-on-cloudflare/blob/main/AGENTS.md. Pick the one recipe whose trigger matches my problem, read it, and apply it to this repo. Run its Verify step and tell me the result. Test anything marked Unverified before relying on it.
```

**By hand.** Start with [`01-start/nextjs-on-workers-opennext.md`](01-start/nextjs-on-workers-opennext.md).

## Moving from Vercel

Read [`migrate-from-vercel/README.md`](migrate-from-vercel/README.md) for the cutover order, then the recipes it links. What gets worse on Workers:

* **Uncached renders are slower.** About 1.8 s on Vercel against 2.7 to 7 s on Workers, and 3 to 4 s when a fresh instance starts. Cache public pages in Workers Cache and most visits take 0.3 to 0.4 s.
* **No `sharp`.** Image processing runs in WASM, which is slower and needs its own setup.
* **Postgres needs Hyperdrive**, a connection per request, and the Worker placed next to the database.
* **No preview URL per branch.** Workers that use Durable Objects get none, and OpenNext's cache uses two. Branches deploy to one shared preview Worker, which has no deployment protection unless you add it.
* **Hard limits:** 64 MiB uncompressed bundle, 128 MB memory per isolate, 1 s startup.
* **`proxy.ts` (middleware) is labelled experimental** in OpenNext.
* **Customers' root domains need CNAME flattening** at their DNS provider. An A record does not work with Cloudflare for SaaS.
* **Stale DNS after the switch.** Some visitors keep reaching Vercel for up to a day or two. Leave the Vercel project running for a few hours after you change nameservers, or they get a 503.

---
## License

Code snippets are under the [MIT License](./LICENSE). The prose is under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).

---

<p align="center">
  <a href="https://www.coduo.co">
    <picture>
      <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/sahildave/coduo-skills/main/assets/apps/coduo-dark.svg">
      <img src="https://raw.githubusercontent.com/sahildave/coduo-skills/main/assets/apps/coduo-light.svg" height="56" alt="coduo">
    </picture>
  </a>
  <br>
  creating software<br>with intention
</p>

<p align="center">
  <a href="https://www.coduo.co"><img src="https://raw.githubusercontent.com/sahildave/coduo-skills/main/assets/apps/skilltopia.png" width="48" alt="skilltopia" title="skilltopia"></a>&nbsp;&nbsp;
  <a href="https://gitsu.app"><img src="https://raw.githubusercontent.com/sahildave/coduo-skills/main/assets/apps/gitsu.png" width="48" alt="gitsu" title="gitsu"></a>&nbsp;&nbsp;
  <a href="https://gitback.us"><img src="https://raw.githubusercontent.com/sahildave/coduo-skills/main/assets/apps/gitback.png" width="48" alt="gitback" title="gitback"></a>&nbsp;&nbsp;
  <a href="https://shareling.app"><img src="https://raw.githubusercontent.com/sahildave/coduo-skills/main/assets/apps/shareling.png" width="48" alt="shareling" title="shareling"></a>&nbsp;&nbsp;
  <a href="https://www.coduo.co"><img src="https://raw.githubusercontent.com/sahildave/coduo-skills/main/assets/apps/macappspace.png" width="48" alt="macapp.space" title="macapp.space"></a>
</p>
