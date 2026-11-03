<p align="center"><img src="banner.png" alt="Next.js on Cloudflare Workers"></p>

# Next.js on Cloudflare Workers: a cookbook for agents

**27 recipes** for deploying and running Next.js 16 apps on Cloudflare Workers with OpenNext (`@opennextjs/cloudflare`). 

### Covers the adapter setup: 
* Postgres through Hyperdrive,
* Workers Cache and purging,
* Cache Components patches,
* Custom domains
* images without `sharp`
* crons
* logs,
* Unrelated tech - Payload CMS, Supabase Auth,
* and an incident playbook.

### Source 
Every recipe comes from three Next.js apps that moved to Workers and run there in production:
* [macapp.space](https://macapp.space) content site with Payload CMS and Supabase
* [Shareling.app](https://shareling.app) web-app companion to a Mac app, with Supabase Auth, Postgres, Stripe.
* [gitback.us](https://gitback.us) A multi-tenant user GitHub-native feedback platform that serves clients and end-users. Clients can have custom domains.

Recipes include the symptoms, the cause and the fix for any quirks.

> Code that was not run in the form shown is marked **Unverified**.

### Last Versions: 

| Package | Version |
|---------|----------------|
| Next|16.3.x                |
| `@opennextjs/cloudflare`    |1.20.15                |
| `wrangler`  | 4.156.2                |
| Node   | 24                |

Last updated: 3 Nov 2026.
Patches target exact versions, so regenerate them against yours.

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

Copy paste this into your agent's prompt:

**As a skill** 

```
Copy the skill from https://github.com/sahildave/nextjs-on-cloudflare/blob/main/SKILL.md to find the one recipe that fits the task.
```

**As a reference** 
```
Read https://github.com/sahildave/nextjs-on-cloudflare/blob/main/AGENTS.md in this cookbook and follow the recipe that matches." It lists a trigger for every recipe and a start-here order for a new app.
```


**By hand** 

Start with `01-start/nextjs-on-workers-opennext.md`. 

## Migration from Vercel

Start with `migrate-from-vercel/nextjs-on-workers-opennext.md` to get the overview of the migration process.

### Gotchas
* 


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
  <a href="https://www.coduo.co"><img src="https://raw.githubusercontent.com/sahildave/coduo-skills/main/assets/apps/futari.png" width="48" alt="futari" title="futari"></a>
</p>
