---
name: morivo-deploy
description: Manages the morivo.cz website on Cloudflare (Workers API + KV content + D1 auth/contact + R2 uploads + Pages SPA). Use whenever a non-coder collaborator wants to deploy a change, edit content, view logs, restore a backup, or troubleshoot the live site. Triggers on /morivo-deploy, "deploy morivo", "morivo logs", "publish morivo", "edit morivo content", "morivo down", or anything else about the morivo site.
---

# Morivo deploy & operations

You are operating the Cloudflare-native deployment of **morivo.cz** for a
non-coder. They typed `/morivo-deploy` (or similar) and expect you to
do the work — explain what you're doing in one short sentence, then act.

## Stack at a glance

| Concern | Service | Binding / name |
|---|---|---|
| Static SPA | Cloudflare Pages | project `morivo-web` |
| HTTP API | Cloudflare Workers | worker `morivo-api` |
| Content (FAQs, testimonials, gallery, …) | Workers KV | `CONTENT_KV` (`morivo-content`) |
| Sessions | Workers KV (TTL) | `SESSION_KV` (`morivo-sessions`) |
| Auth users + contact submissions | D1 (SQLite) | `MORIVO_DB` (`morivo`) |
| Image uploads | R2 | bucket `morivo-uploads` |
| Video hosting | Cloudflare Stream | unchanged |
| Email | Resend | unchanged |

Live URLs after cutover: `morivo.cz` (Pages), `api.morivo.cz` (Workers),
`admin.morivo.cz` (same Pages bundle, admin routes).

The repo lives at `https://github.com/MrSucik/morivo-hyperboost`. Default
local clone: `~/p/morivo-hyperboost`.

## Safety contract

These guarantees apply to every operation in this skill. Do not strip
them out when editing.

- **Every code-deploying command** (preview deploy, prod deploy,
  redeploy, publish, recover-and-redeploy) **must** run the
  [Pre-deploy git safety](#pre-deploy-git-safety-mandatory) gate first.
  Read-only commands (logs, admin UI, password reset, backup restore)
  are exempt.
- **Every prod deploy** must end with the
  [Post-deploy bundle smoke](#post-deploy-bundle-smoke). If the marker
  check fails, surface "rollback recommended — bundle does not match
  the deployed branch" and stop. Do not claim success.
- **One code path to prod.** There is exactly one way the skill ships
  code to morivo.cz — through the gate above. Don't add a shortcut
  that bypasses it.

## First-time setup (once per machine)

Detect first-time state by checking if `~/p/morivo-hyperboost` exists
*and* the `CLOUDFLARE_API_TOKEN` env var is set in the user's shell.
If either is missing, walk through these steps in order:

1. **Clone the repo** (if missing):
   ```bash
   mkdir -p ~/p && cd ~/p && gh repo clone MrSucik/morivo-hyperboost
   ```
   If `gh` isn't installed, fall back to `git clone https://github.com/MrSucik/morivo-hyperboost.git`.

2. **Install dependencies**:
   ```bash
   cd ~/p/morivo-hyperboost/apps/api && npm install --ignore-scripts --no-audit
   cd ~/p/morivo-hyperboost/apps/web && npm install --ignore-scripts --no-audit
   ```

3. **Set the Cloudflare API token** — the maintainer issued you an
   account-scoped API token. Add it to your shell startup so wrangler
   picks it up automatically:
   ```bash
   # Add to ~/.zshrc (or ~/.bashrc):
   export CLOUDFLARE_API_TOKEN="<paste-token-here>"
   export CLOUDFLARE_ACCOUNT_ID="1e9a68c75a60b5e5509ed8a2a0f12af5"
   ```
   Then reload: `source ~/.zshrc`. Verify with `npx wrangler whoami` —
   you should see "Morivo" as the account.

That's it. The skill assumes the CF account already owns the bindings
(KV, D1, R2, Pages). To re-provision from scratch, see "Recover from a
fresh CF account" at the bottom.

The Cloudflare account that hosts everything is **Morivo**
(account ID `1e9a68c75a60b5e5509ed8a2a0f12af5`). You should never need
to touch the Cloudflare dashboard — every operation below works via
the API token.

## Pre-deploy git safety (mandatory)

**Run this BEFORE every code-deploying command** (preview deploy, prod
deploy, redeploy, publish). Skip only for read-only operations (logs,
content edits via admin UI, password reset, backup restore).

On 2026-05-12 this skill shipped a stale local clone to prod and
rolled the live site back ~6 days. The local `master` was behind
`origin/master`, the operator never ran `git fetch`, and a leftover
`apps/web/dist/` from an older build was re-deployed verbatim. These
gates exist to make that impossible.

Start from a known-good state:

```bash
cd ~/p/morivo-hyperboost
git fetch origin
```

Then walk through every gate below in order. **Abort** on any failure
and surface the reason to the operator — never auto-resolve dirty
state or auto-discard work.

1. **Working tree clean?** `git status --porcelain` must be empty. If
   not, print the dirty files and ask the operator whether to commit,
   stash, or discard before continuing. Do not auto-stash.
2. **On `master`?** `git rev-parse --abbrev-ref HEAD` must equal
   `master`. If not, print the current branch and tell the operator
   to `git checkout master` first.
3. **Local in sync with `origin/master`?** Compare
   `git rev-parse HEAD` and `git rev-parse origin/master`:
   - **Behind** (`git rev-list --count HEAD..origin/master` > 0):
     say explicitly "your local is N commits behind origin/master —
     the deploy would publish stale code", show
     `git log --oneline HEAD..origin/master`, confirm with the
     operator, then run `git pull --ff-only origin master`.
   - **Ahead** (`git rev-list --count origin/master..HEAD` > 0):
     print the unpushed commits with
     `git log --oneline origin/master..HEAD`, **abort**, ask the
     operator to push or reset first.
   - **Diverged** (both counts > 0): **abort** with instructions to
     investigate manually. Do not attempt to rebase or merge from
     the skill.
4. **Show what's about to ship.** Print `git log -1 --oneline` so the
   operator sees the head commit before any artifacts are built.
5. **Nuke stale build artifacts.** Always run, no exceptions:
   ```bash
   rm -rf apps/web/dist apps/api/.wrangler
   ```
   A stale `dist/` was the proximate cause of the 2026-05-12
   regression — the build appeared to succeed because old artifacts
   were still on disk.
6. **Install deps from scratch.**
   ```bash
   cd apps/web && npm install --prefer-offline --no-audit
   ```
   (Do the same for `apps/api` when deploying the Worker.)

Only after **all six** gates pass may you proceed to `wrangler ... deploy`.

## Day-to-day operations

### Test changes safely in the preview environment FIRST

A separate preview environment exists with its own KV/D1/R2/Worker/Pages,
seeded with the same content as prod. Use it before any prod deploy.

**First**: run the
[Pre-deploy git safety](#pre-deploy-git-safety-mandatory) gate —
preview deploys are still code deploys and the same staleness footgun
applies.

```bash
# Deploy API to preview Worker
cd ~/p/morivo-hyperboost/apps/api
npx wrangler deploy --env=preview
# → https://morivo-api-preview.morivo.workers.dev

# Deploy SPA to preview Pages
cd ~/p/morivo-hyperboost/apps/web
VITE_API_URL=https://morivo-api-preview.morivo.workers.dev npm run build
COMMIT=$(git -C ~/p/morivo-hyperboost rev-parse --short HEAD)
npx wrangler pages deploy ./dist \
  --project-name=morivo-web-preview \
  --branch=master \
  --commit-message="preview ${COMMIT} by ${USER}"
# → https://morivo-web-preview.pages.dev
```

Smoke-test there. Once happy, deploy to prod with the steps below.

### Deploy a code change

**First**: run the
[Pre-deploy git safety](#pre-deploy-git-safety-mandatory) gate. Do not
skip it — that's what shipped a stale clone to prod on 2026-05-12.

1. Deploy the API (Workers), if it changed:
   ```bash
   cd ~/p/morivo-hyperboost/apps/api && npx wrangler deploy
   ```
2. Build and deploy the SPA (Pages):
   ```bash
   cd ~/p/morivo-hyperboost/apps/web
   VITE_API_URL=https://api.morivo.cz npm run build
   COMMIT=$(git -C ~/p/morivo-hyperboost rev-parse --short HEAD)
   npx wrangler pages deploy ./dist \
     --project-name=morivo-web \
     --branch=master \
     --commit-message="deploy ${COMMIT} by ${USER}"
   ```
3. Run the [Post-deploy bundle smoke](#post-deploy-bundle-smoke).
   **Stop here if it fails** and surface the rollback recommendation.
4. Verify endpoints:
   ```bash
   curl -s https://api.morivo.cz/health
   curl -sI https://morivo.cz/ | head -1
   ```

If the user only edited content in the admin UI (not code), they don't
need to deploy anything — KV writes propagate within ~60 seconds.

### Post-deploy bundle smoke

After every prod Pages deploy, verify the served bundle matches the
branch you just shipped. The check fetches the homepage, finds the
SPA bundle reference, and greps for a stable marker present in every
build of the current SPA (`morivo:cookie-consent`).

```bash
HTML=$(curl -s https://morivo.cz/)
ASSET=$(printf '%s' "$HTML" | grep -oE '/assets/index-[A-Za-z0-9_-]+\.js' | head -1)
if [ -z "$ASSET" ]; then
  echo "❌ rollback recommended — could not find SPA bundle reference in served HTML"
  exit 1
fi
if ! curl -s "https://morivo.cz${ASSET}" | grep -q "morivo:cookie-consent"; then
  echo "❌ rollback recommended — bundle does not match the deployed branch (marker missing)"
  exit 1
fi
echo "✅ smoke: prod is serving the expected bundle (${ASSET})"
```

If the marker is missing, page the operator with the rollback
recommendation. Don't claim the deploy succeeded — Cloudflare Pages
retains the previous deployment and rollback is a single dashboard
click (or `wrangler pages deployment` rollback flow).

### View live logs

```bash
cd ~/p/morivo-hyperboost/apps/api && npx wrangler tail morivo-api --format=pretty
```
Press `Ctrl-C` to stop.

### Edit content via the admin UI

Open `https://admin.morivo.cz` (or `https://morivo.cz/admin`), log in
with the admin email + password. The skill should NOT ask for the
password — the user knows it. If they've lost it, use the recovery
path below.

### Reset a forgotten admin password

1. Generate a bcrypt hash for the new password (locally — never type
   the cleartext into any chat):
   ```bash
   node -e "const b=require('bcryptjs');b.hash(process.env.NEWPASS,10).then(h=>console.log(h))" \
     # paste with NEWPASS=... in front, e.g. via `read -s NEWPASS && export NEWPASS`
   ```
2. Update D1:
   ```bash
   cd ~/p/morivo-hyperboost/apps/api
   npx wrangler d1 execute morivo --remote --command="UPDATE account SET password='<the-bcrypt-hash>' WHERE provider_id='credential'"
   ```

### Tail Sentry errors

The Worker emits errors to Sentry once `SENTRY_DSN` is set. Check the
Sentry project at `https://morivocz.sentry.io/` — issues prefixed
`MORIVOCZ-WEB-*` are SPA, `MORIVOCZ-API-*` are the Worker.

Many `window.webkit.messageHandlers` errors are noise from in-app
browsers (Instagram, Facebook). Ignore unless they're a real bundle
stack trace.

## Restore from a backup

Backups live at `~/p/morivo-backups/<date>/` on the maintainer's machine.
Each contains:

- `morivo-postgres-full.sql.gz` — historic Postgres dump (legacy)
- `tables/<table>.json` — per-table JSON for KV/D1 import
- `morivo-uploads.tar.gz` — R2 image tarball
- `MANIFEST.sha256` — checksums

To re-run the import:
```bash
cd ~/p/morivo-hyperboost
node scripts/migrate-pg-to-cf.mjs --backup=/path/to/backup --only=kv,d1,r2
```

Use `--dry-run` first if unsure.

## Recover from a fresh CF account

If the Cloudflare account is lost and you need to start over with a
new one (or re-provision the existing Morivo account from scratch),
re-run the full provisioning:

```bash
cd ~/p/morivo-hyperboost
export CLOUDFLARE_API_TOKEN="<new-token>"
export CLOUDFLARE_ACCOUNT_ID="<new-account-id>"
python3 scripts/cf-provision.py
```

This idempotently creates KV namespaces, the D1 database, the R2
bucket, applies migrations, and patches `apps/api/wrangler.toml` with
the new IDs. After it finishes, set the secrets and re-import data:

```bash
cd ~/p/morivo-hyperboost/apps/api
echo $BETTER_AUTH_SECRET | npx wrangler secret put BETTER_AUTH_SECRET
echo $RESEND_API_KEY | npx wrangler secret put RESEND_API_KEY
# … repeat for NOTIFICATION_EMAIL, CLOUDFLARE_ACCOUNT_ID,
#    CLOUDFLARE_STREAM_API_TOKEN, SENTRY_DSN

cd ~/p/morivo-hyperboost && node scripts/migrate-pg-to-cf.mjs --backup=<latest-backup>
```

Then re-deploy the Worker and Pages as in "Deploy a code change" —
the [Pre-deploy git safety](#pre-deploy-git-safety-mandatory) gate
applies here too, and the
[Post-deploy bundle smoke](#post-deploy-bundle-smoke) must pass
before you swap nameservers.

Finally, swap nameservers at the registrar to the new account's
nameservers (visible in the Cloudflare dashboard under the zone
overview).

## DNS — already wired

DNS for `morivo.cz`, `www.morivo.cz`, `admin.morivo.cz`, and
`api.morivo.cz` is already configured on the Cloudflare zone in the
**Morivo** account, and the custom domain attachments to the Pages
project (`morivo-web`) and Worker (`morivo-api`) are in place.
You shouldn't need to touch DNS.

If a record needs editing (e.g. adding an SPF/DMARC TXT for a new
sender), use the wrangler CLI rather than the dashboard:
```bash
# List records
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/zones/3d5cdd35172af7c1fd539fe1b28d3ac8/dns_records"
```

## Safety rules

1. **Never `git push --force` to `master`** — the production deploy
   trigger lives there.
2. **Never delete a KV namespace, D1 database, or R2 bucket without
   confirming with the user** — the data isn't recoverable from CF
   alone (only from local backups).
3. **Before any destructive D1 query, take a snapshot first**:
   ```bash
   npx wrangler d1 export morivo --remote --output=morivo-$(date +%Y-%m-%d).sql
   ```
4. **Don't share the `CLOUDFLARE_API_TOKEN`.** It grants full control
   of the Morivo Cloudflare account. If it leaks, ask the maintainer
   to rotate it via the dashboard.
5. **Never bypass the
   [Pre-deploy git safety](#pre-deploy-git-safety-mandatory) gate or
   the [Post-deploy bundle smoke](#post-deploy-bundle-smoke).** These
   exist because past deploys silently rolled the site back. If you
   need to "just push something quickly", that's exactly the case the
   gate is for.

## When something is broken

Walk through this in order, top to bottom, stopping when one thing
fixes it:

1. `curl https://api.morivo.cz/health` — does it 200?
2. `npx wrangler tail morivo-api --format=pretty` — see live errors
3. `curl https://morivo.cz/` — does it 200?
4. Open browser DevTools console on `https://morivo.cz/` — any CSP
   violations or red errors?
5. Check Sentry: `https://morivocz.sentry.io/`
6. `npx wrangler d1 execute morivo --remote --command="SELECT count(*) FROM contact_submissions"` — D1 reachable?
7. `npx wrangler kv key get "content:faqs" --binding=CONTENT_KV --remote --preview false` — KV reachable?
8. If still stuck, restore from the latest backup as documented above.
