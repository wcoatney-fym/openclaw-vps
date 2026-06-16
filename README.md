# openclaw-vps

Operational notes and troubleshooting for the **OpenClaw** agent the FYM team runs on a Hostinger VPS.

> ⚠️ **No secrets in this repo.** Tokens/keys live only in the VPS config (`/data/.openclaw/openclaw.json`) and the container `.env`. The committed config (`config/openclaw.json.redacted`) has all secret-bearing fields replaced with `<REDACTED>`. Keep it that way — see `.gitignore`.

## Architecture

| Piece | Detail |
|---|---|
| Host | `srv1759405.hstgr.cloud` (`root@2.25.208.160`), Ubuntu 24.04, Hostinger KVM 2 |
| Runtime | Docker + Traefik (reverse proxy, Let's Encrypt) |
| Container | `openclaw-lynj-openclaw-1` from `ghcr.io/hostinger/hvps-openclaw:latest` |
| Process | `node server.mjs`, runs as user **`node` (uid 1000)** |
| Config (host) | `/docker/openclaw-lynj/data/.openclaw/openclaw.json` |
| Config (in container) | `~/.openclaw/openclaw.json` (HOME=`/data`) |
| Compose | `/docker/openclaw-lynj/docker-compose.yml` (+ `env_file: .env`) |
| Ports | proxy `43942`; gateway control UI `18789` (loopback) |
| Gateway log | inside container: `/tmp/openclaw-1000/openclaw-YYYY-MM-DD.log` (JSON lines) |

## Access

SSH (dedicated key):

```bash
ssh -i ~/.ssh/openclaw_vps root@2.25.208.160
```

Run the OpenClaw CLI as the app user:

```bash
docker exec -u node openclaw-lynj-openclaw-1 openclaw <command>
# e.g.
docker exec -u node openclaw-lynj-openclaw-1 openclaw config validate
docker exec -u node openclaw-lynj-openclaw-1 openclaw channels list --all
docker exec -u node openclaw-lynj-openclaw-1 openclaw channels status --probe --json
docker exec -u node openclaw-lynj-openclaw-1 openclaw plugins list
docker exec -u node openclaw-lynj-openclaw-1 openclaw doctor --lint
```

Recreate / restart the stack:

```bash
cd /docker/openclaw-lynj && docker compose up -d
```

## Current status (2026-06-15)

- ✅ Gateway **healthy** (`running`, config valid).
- ✅ **Slack fully working** (Socket Mode) → workspace *Team FYM*, bot `crmteambot`, channel `#crm-openclaw` (`C0BAK76B47M`). Bot replies to @mentions; survives cold restart; status `running/connected/healthy`.

**Two fixes were needed (full write-up: [docs/slack-troubleshooting.md](docs/slack-troubleshooting.md)):**
1. **Enable the channel plugin** — `plugins.entries.slack.enabled: true` (`openclaw plugins enable slack`). The third-party `@openclaw/slack` plugin doesn't auto-load; without this the channel never materializes.
2. **Fix the channel allowlist key** — under `groupPolicy: allowlist`, the entry must be keyed by channel **id** (`C0BAK76B47M`) or bare name (`crm-openclaw`), **not** the `#`-prefixed display name. A `#`-prefixed key matches nothing at message time, so the gateway silently drops every message (`matchKey=none`).

## Repo layout

```
config/openclaw.json.redacted      # current live config, secrets stripped
docs/incident-2026-06-15-config-permissions.md
docs/slack-troubleshooting.md      # repro write-up for Hostinger support
```

## Security notes

- The agent is configured with `tools.profile: full`, `commands.bash: true`, `tools.elevated.enabled: true` (model Opus). Once Slack connects, **anyone who can post in `#crm-openclaw` can run shell commands on this VPS.** The channel allowlist (`groupPolicy: allowlist`) is the only guardrail.
- `openclaw doctor` flags that `openclaw.json` stores tokens in plaintext. Consider migrating to SecretRefs (`openclaw secrets configure`).
