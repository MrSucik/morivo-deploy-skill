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

Then re-deploy the Worker and Pages as in "Deploy a code change".
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
