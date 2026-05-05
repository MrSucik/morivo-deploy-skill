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

## First-time setup (once per machine)

Detect first-time state by checking if `~/p/morivo-hyperboost` exists
*and* `npx wrangler whoami` returns a logged-in account. If either is
missing, walk through these steps in order:

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

3. **Authenticate with Cloudflare** — interactive, ask the user to
   paste this in their terminal so the OAuth callback can hit
   localhost:
   ```
   ! cd ~/p/morivo-hyperboost/apps/api && npx wrangler login
   ```

That's it. The skill assumes the CF account already owns the bindings
(KV, D1, R2, Pages). To re-provision from scratch, see "Recover from a
fresh CF account" at the bottom.

## Day-to-day operations

### Deploy a code change

1. Pull latest, install deps if `package-lock.json` changed:
   ```bash
   cd ~/p/morivo-hyperboost && git pull --ff-only
   ```
2. Deploy the API (Workers):
   ```bash
   cd ~/p/morivo-hyperboost/apps/api && npx wrangler deploy
   ```
3. Deploy the SPA (Pages) — set `VITE_API_URL` to the production API:
   ```bash
   cd ~/p/morivo-hyperboost/apps/web
   VITE_API_URL=https://api.morivo.cz npm run build
   npx wrangler pages deploy ./dist --project-name=morivo-web --branch=master
   ```
4. Smoke-test:
   ```bash
   curl -s https://api.morivo.cz/health
   curl -sI https://morivo.cz/ | head -1
   ```

If the user only edited content in the admin UI (not code), they don't
need to deploy anything — KV writes propagate within ~60 seconds.

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

Backups live at `/Users/xxx/p/morivo-backups/<date>/` on the
maintainer's machine. Each contains:

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

If the user lost access to the Cloudflare account and started a new
one, re-run the full provisioning:

```bash
cd ~/p/morivo-hyperboost
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

Then re-deploy the Worker and Pages as in "Deploy a code change".

## DNS swap (one-time)

If `morivo.cz` doesn't yet point at Pages and `api.morivo.cz` doesn't
point at the Worker, attach the custom domains in the Cloudflare
dashboard:

- **Pages → Custom domains**: add `morivo.cz` and `www.morivo.cz`,
  also `admin.morivo.cz`.
- **Workers → Settings → Triggers → Custom Domains**: add
  `api.morivo.cz`.

Cloudflare will create the DNS records automatically if the zone is
on the same account.

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
4. **The Coolify VPS still hosts the legacy Postgres copy** until
   the maintainer manually retires it (per the migration plan,
   7-day soak after cutover). Don't stop those containers without
   explicit confirmation.

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
