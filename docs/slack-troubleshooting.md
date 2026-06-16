# Slack channel will not connect — troubleshooting / support repro

**Image:** `ghcr.io/hostinger/hvps-openclaw:latest` (OpenClaw `2026.5.28`, commit `e932160`)
**Date:** 2026-06-15

## Summary
A Slack channel that is **correctly configured, enabled, with valid Socket Mode tokens, and whose plugin is loaded and healthy** is **never materialized by the gateway** at startup. No error, no skip reason — the channel simply doesn't exist at runtime.

## What is verified correct
| Check | Command | Result |
|---|---|---|
| Config valid | `openclaw config validate` | `Config valid` |
| Channel enabled | `openclaw config get channels.slack.enabled` | `true` |
| Mode | `openclaw config get channels.slack.mode` | `socket` |
| Bot token | Slack `auth.test` with the `xoxb-` token | `ok:true`, team *Team FYM*, user `crmteambot` |
| App token | `openclaw config get channels.slack.appToken` | valid `xapp-1-…` (98 chars) |
| Plugin loaded | `openclaw plugins inspect slack` | `Status: loaded`, `Capabilities: channel: slack` |
| Plugin issues | `openclaw plugins doctor` | `No plugin issues detected.` |

Config shape matches the documented Socket Mode example
(`channels.slack` = `{ enabled:true, mode:"socket", botToken, appToken }`; single default account, no `accounts.*`).

## The anomaly
```bash
openclaw channels list
# -> no configured chat channels

openclaw channels list --all
# -> Slack: installed, not configured, disabled      <-- but enabled:true is set

openclaw channels status --probe --json
# -> "channels": {}, "channelAccounts": {}, "channelOrder": []   (empty)
```

Debug startup (`OPENCLAW_LOG_LEVEL=debug`) around channel start shows **no Slack handling at all** — other channels are explicitly skipped with reasons, Slack is absent:
```
[gateway] starting channels and sidecars...
[plugins] [hooks] running gateway_start (1 handlers)
[cron] cron: started
[ws] webchat connected ...
# (no slack / no channel materialization, no skip reason)
INFO: Plugin "telegram" does not meet requirements, skipping
INFO: Plugin "whatsapp" does not meet requirements, skipping
# slack is NOT listed as skipped — it meets requirements but is never started
```

## Interpretation
The gateway's runtime-config builder appears to **silently drop the configured + enabled + plugin-loaded Slack channel** when materializing channels — even though the CLI config reader (`config get`) sees it and validation passes. This looks like a bug or an undocumented required step in the `hvps-openclaw` image.

## Things already tried (no effect)
- `openclaw channels add --channel slack --bot-token … --app-token …` → "Added Slack account default", but `channels list` still empty.
- Replaced the bad `appToken` (was a Slack **App ID**, not `xapp-`) with a valid app-level token.
- Multiple full container restarts / `docker compose up -d` recreate (confirmed the node process cycles and reloads current config).
- `openclaw channels login --channel slack` → "Channel slack does not support login" (expected for Socket Mode).
- Confirmed no stale runtime-snapshot/registry files in `/data/.openclaw`.

## Open questions for Hostinger / OpenClaw
1. Is there an additional step beyond `channels.slack.enabled:true` + tokens to get the gateway to **materialize** a Socket Mode channel in this image?
2. Why is an enabled, plugin-loaded channel reported as `installed, not configured, disabled` and excluded from `channels status` with no diagnostic?

## Slack app settings to double-check (config side is fine)
- Socket Mode enabled; app-level token has `connections:write`.
- Bot scopes: `app_mentions:read`, `channels:read`/`history`, `chat:write`, `commands`, `im:*`, `groups:*`.
- Event subscriptions: `app_mention`, `message.channels`, `message.im`, `message.groups`.
- Bot invited to `#crm-openclaw` (`/invite @crmteambot`).
