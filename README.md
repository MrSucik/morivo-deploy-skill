# morivo-deploy — Claude Code skill

A Claude Code skill that automates day-to-day operations for the
[morivo.cz](https://morivo.cz) website on its Cloudflare-native stack
(Workers + KV + D1 + R2 + Pages).

## What it does

Drop in a chat with Claude Code and type `/morivo-deploy` (or just
"deploy morivo", "show morivo logs", "edit morivo content", "restore
backup"). The skill walks you through the right command for each
case, runs it for you, and confirms the result.

Designed for non-coder collaborators who manage content but don't
want to memorise the wrangler CLI.

## Install

```bash
# Inside Claude Code
/plugin install MrSucik/morivo-deploy-skill
```

Or clone manually:

```bash
git clone https://github.com/MrSucik/morivo-deploy-skill ~/.claude/plugins/morivo-deploy-skill
```

Then restart Claude Code.

## First-time setup

The skill itself walks you through this on the first invocation —
clone the [morivo-hyperboost](https://github.com/MrSucik/morivo-hyperboost)
repo, install dependencies, and add the Cloudflare API token (the
maintainer issues you a token scoped to the dedicated `Morivo`
Cloudflare account):

```bash
export CLOUDFLARE_API_TOKEN="<token-from-maintainer>"
export CLOUDFLARE_ACCOUNT_ID="1e9a68c75a60b5e5509ed8a2a0f12af5"
```

Add those two lines to `~/.zshrc` (or `~/.bashrc`) so wrangler picks
them up on every shell.

## What's covered

- Deploy a code change (API → Workers, SPA → Pages)
- View live logs
- Edit content via the admin UI
- Reset a forgotten admin password
- Restore from a local backup
- Recover from a fresh CF account
- Custom domain / DNS swap
- Triage when something is broken

## License

MIT — see [LICENSE](LICENSE).
