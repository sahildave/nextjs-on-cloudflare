---
title: Secrets and env vars on Cloudflare Workers
description: "Split env into build-time keys (CI env for next build) and runtime keys (Worker secrets), push runtime secrets with wrangler secret bulk from any secret manager without writing a value to disk or the log, and keep plain vars out of the secret set."
tags: [cookbook, cloudflare]
topics: [cloudflare, secrets, env, wrangler, github-actions]
date: 2026-11-03
path: 01-start/secrets-and-env.md
---
# Secrets and env vars on Cloudflare Workers

A Next.js app on Workers reads env in two places that never see each other. `next build` runs in CI and reads `NEXT_PUBLIC_*` plus any server key it touches while prerendering. The Worker reads everything else from `env` at runtime, as Worker secrets or plain vars. This recipe sets up both paths from one secret store: a CI step exports the build keys, and a script pipes the runtime keys the app actually reads into `wrangler secret bulk`. The examples call a placeholder `secret-export` command; replace it with your manager's JSON export (1Password, Doppler, Vault, or a local file for small projects). The shape stays.

## Prerequisites

- A Worker deployed per [./nextjs-on-workers-opennext.md](./nextjs-on-workers-opennext.md).
- The deploy workflow from [./deploy-from-github-actions.md](./deploy-from-github-actions.md).
- Tools: `jq`, `wrangler` auth (`wrangler login` or `CLOUDFLARE_API_TOKEN`), and your secret manager's CLI.
- A way for CI to read the store. Prefer an OIDC login from GitHub Actions (no stored secret) if your manager supports it; otherwise one GitHub secret holding a read-only token.

## Steps

1. Pick one place where secrets are set, and set them only there. If a value is wrong, fix it in the store and push again; a value typed into the Cloudflare dashboard drifts from the store.
2. Split keys by when they are read.
   - Build time (CI env): `NEXT_PUBLIC_*` (inlined into the client bundle), plus the server keys `next build` touches while prerendering. A Payload app needed `PAYLOAD_SECRET`, `DATABASE_URI`, `DATABASE_URL` and the media storage keys. Without the storage keys, prerendered pages baked local media URLs instead of the CDN.
   - Runtime (Worker secrets): everything else the code reads from `env`. Never `NEXT_PUBLIC_*`.
   - Some keys need both.
3. One-off secrets need no tooling: `pnpm exec wrangler secret put CRON_SECRET` prompts for the value. Use the script below once there are more than a handful.
4. Add the CI export step (Code below) before the Build step.
5. Add `scripts/push-cloudflare-secrets.sh` (Code below) and run it by hand whenever a runtime secret changes. It reads the source for `env.KEY` and `env["KEY"]`, so only keys the app uses are pushed.
6. Keep plain vars (`APP_ENV`, `GIT_SHA`) out of the push. A secret and a var of the same name clash. The script skips them via a `VARS` list.

## Code

### CI step: export build env (in `.github/workflows/cloudflare-deploy.yml`)

Replace the `secret-export` line with your manager's command. It must print a JSON array of `{ "key": ..., "value": ... }` objects, or adjust the first `jq` to your format. If your store has a shared path and an app path, export both in that order so the app's value wins.

```yaml
      - name: Export build env from the secret store
        run: |
          secret-export --env="$APP_ENV" --format=json > "$RUNNER_TEMP/secrets.json"
          # Keep NEXT_PUBLIC_* plus the server keys the build needs.
          jq -s 'add | map(select((.key | startswith("NEXT_PUBLIC_")) or .key == "PAYLOAD_SECRET" or .key == "DATABASE_URI") | {(.key): .value}) | add' "$RUNNER_TEMP/secrets.json" > "$RUNNER_TEMP/build-env.json"
          # Mask every value (and each line of a multi-line one) in the log.
          while IFS= read -r v; do
            [ -n "$v" ] || continue
            while IFS= read -r line; do [ -n "$line" ] && echo "::add-mask::$line"; done <<<"$v"
          done < <(jq -r 'to_entries[] | .value' "$RUNNER_TEMP/build-env.json")
          # Write to $GITHUB_ENV with a random heredoc delimiter, so a value cannot inject a variable.
          delim="EOF_$(openssl rand -hex 16)"
          jq -r 'to_entries[] | .key' "$RUNNER_TEMP/build-env.json" | while IFS= read -r k; do
            { echo "$k<<$delim"; jq -r --arg k "$k" '.[$k]' "$RUNNER_TEMP/build-env.json"; echo "$delim"; } >> "$GITHUB_ENV"
          done
          echo "Exported for $APP_ENV: $(jq -r 'keys | join(", ")' "$RUNNER_TEMP/build-env.json")"
          rm -f "$RUNNER_TEMP/secrets.json" "$RUNNER_TEMP/build-env.json"

      - name: Require build keys
        run: |
          bad=
          for k in NEXT_PUBLIC_APP_URL; do
            [ -n "${!k-}" ] || bad="$bad $k"
          done
          [ -z "$bad" ] || { echo "::error::Build needs a value for:$bad."; exit 1; }
```

### `scripts/push-cloudflare-secrets.sh`

```bash
#!/usr/bin/env bash
# Pushes this app's runtime secrets from the secret store to the Worker. Prints key names only.
#
#   scripts/push-cloudflare-secrets.sh --dry-run        # list the keys (env prod)
#   scripts/push-cloudflare-secrets.sh                  # push env prod
#   scripts/push-cloudflare-secrets.sh dev [--dry-run]  # another store env
set -euo pipefail

# Plain vars in wrangler.jsonc or set per deploy; a secret of the same name clashes.
VARS=(APP_ENV GIT_SHA)

store_env=prod
dry_run=0
for arg in "$@"; do
  case "$arg" in
    --dry-run) dry_run=1 ;;
    -*) echo "usage: $0 [store-env] [--dry-run]" >&2; exit 2 ;;
    *) store_env=$arg ;;
  esac
done

repo=$(cd "$(dirname "$0")/.." && pwd)

# Held in memory, never on disk. Replace secret-export with your manager's JSON
# export ([{key, value}, ...]); concatenate several exports for layered paths,
# the later one wins.
all=$(
  secret-export --env="$store_env" --format=json |
    jq -s 'add | map({(.key): .value}) | add // {}'
)

# The keys the app reads: `env.KEY` or `env["KEY"]` in src/, the next configs and
# worker.ts, tests excluded.
read_keys=$(
  grep -rhoE --include='*.ts' --include='*.tsx' --include='*.mjs' --exclude='*.test.*' \
    '\benv(\.[A-Z][A-Z0-9_]*|\[["'"'"'][A-Z][A-Z0-9_]*["'"'"']\])' \
    "$repo/src" "$repo/next.config.mjs" "$repo/worker.ts" |
    grep -oE '[A-Z][A-Z0-9_]*' | sort -u
)
keys=()
while IFS= read -r key; do
  case "$key" in NEXT_PUBLIC_*) continue ;; esac
  for var in "${VARS[@]}"; do [ "$key" = "$var" ] && continue 2; done
  if grep -qxF -- "$key" <<<"$read_keys"; then
    keys+=("$key")
  fi
done < <(printf '%s' "$all" | jq -r 'keys[]')

if [ "${#keys[@]}" -eq 0 ]; then
  echo "no stored runtime key is read by the app; nothing to push" >&2
  exit 1
fi

if [ "$dry_run" -eq 1 ]; then
  echo "would push to the Worker (store env $store_env):"
  for key in "${keys[@]}"; do echo "  $key"; done
  exit 0
fi

# Values go from jq straight to wrangler's stdin; no file holds them.
keys_json=$(printf '%s\n' "${keys[@]}" | jq -R . | jq -s .)
printf '%s' "$all" | jq --argjson keys "$keys_json" 'with_entries(select(.key as $k | $keys | index($k)))' |
  (cd "$repo" && pnpm exec wrangler secret bulk >/dev/null)
for key in "${keys[@]}"; do echo "pushed $key"; done
```

If you set `name` in `wrangler.jsonc`, `wrangler secret bulk` picks the Worker from it. Add `--name` for a different Worker. Add `--env` if you use wrangler environments.

## Verify

```bash
scripts/push-cloudflare-secrets.sh --dry-run
# would push to the Worker (store env prod):
#   CRON_SECRET
#   DATABASE_URL
#   ...

scripts/push-cloudflare-secrets.sh
# pushed CRON_SECRET
# pushed DATABASE_URL

pnpm exec wrangler secret list      # names only; the keys you just pushed appear
```

In CI, the export step logs `Exported for production: DATABASE_URI, NEXT_PUBLIC_..., PAYLOAD_SECRET` and no values.

## Gotchas

- The app starts but a feature is "not configured" → the key is in the store but the code reads it some way the grep misses (a destructured `process.env`, a variable key) → read it as `process.env.KEY` or `env.KEY` literally, or add it to the script by hand.
- Deploy fails or the secret is ignored → a key of the same name is also a `vars` entry in `wrangler.jsonc` → keep `APP_ENV` and `GIT_SHA` in `VARS` so the script skips them.
- Every secret push appears as an "unexpected deploy" in the dashboard → each `secret bulk` creates a new Worker version → expected.
- Prerendered pages contain local media URLs → media storage keys were not in the build export → add them to the `jq` select list.
- `NEXT_PUBLIC_*` set only as a Worker secret has no effect on the browser → those are inlined at build → they must be in the CI export, not the push.
- A multi-line value (a private key) breaks `$GITHUB_ENV` → no delimiter → use the random heredoc delimiter shown above, and mask each line.
- A Worker secret is not visible to `next build` → CI and Worker are separate → build keys go through `$GITHUB_ENV`, runtime keys through `wrangler secret bulk`. Some keys need both.
- Changing `CRON_SECRET` or any runtime key needs no rebuild, but the new version takes effect only after the push finishes.
- A key deleted from the store keeps working on the Worker → `secret bulk` and `--secrets-file` only add and update → `wrangler secret delete <KEY>` (`-e preview` for a preview Worker).

## Variant: secrets in the deploy step

You can push runtime secrets on every deploy instead of by hand. `opennextjs-cloudflare deploy` passes flags it does not know through to `wrangler deploy`, which takes `--secrets-file`. The file holds a JSON object of key to value. Write it under `umask 077`, leave out the var-type keys, and delete it in an `if: always()` step.

> **Unverified.** One app ran this shape in production; the steps below were written for this cookbook from its description, not copied from that workflow. Run it once on a preview Worker first.

```yaml
      - name: Write runtime secrets file
        run: |
          umask 077
          # Everything except NEXT_PUBLIC_* (build-time) and the plain vars.
          secret-export --env="$APP_ENV" --format=json |
            jq -s 'add | map(select(.key | test("^(NEXT_PUBLIC_|APP_ENV$|GIT_SHA$)") | not) | {(.key): .value}) | add // {}' \
            > "$RUNNER_TEMP/runtime-secrets.json"
          echo "Runtime keys: $(jq -r 'keys | join(", ")' "$RUNNER_TEMP/runtime-secrets.json")"

      - name: Deploy production
        if: github.ref == 'refs/heads/main'
        run: pnpm exec opennextjs-cloudflare deploy --secrets-file="$RUNNER_TEMP/runtime-secrets.json" --var=APP_ENV:production --var=GIT_SHA:"$GIT_SHA"

      - name: Remove runtime secrets file
        if: always()
        run: rm -f "$RUNNER_TEMP/runtime-secrets.json"
```

This pushes every store key, not only the ones the source reads. To narrow it, filter by an explicit allow-list of key names in the `jq` select. `--secrets-file` adds and updates only; a key removed from the store stays on the Worker until `wrangler secret delete`.
