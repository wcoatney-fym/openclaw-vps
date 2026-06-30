# Incident: gateway crash-loop from invalid `llama-cpp` config key (2026-06-30)

## Symptom
OpenClaw gateway crash-looped on every start; the Hostinger "Starting OpenClaw"
onboarding screen hung indefinitely on *"Please wait while we set up your environment."*

```
[gateway] Gateway failed to start: Invalid config at /data/.openclaw/openclaw.json.
plugins.entries.llama-cpp: Invalid input
[plugins] memory-core: dreaming cron reconcile failed: Invalid config ...
  - plugins.entries.llama-cpp: Unrecognized key: "activation"
WARN: OpenClaw exited with code 1
WARN: OpenClaw process exited before device approval could complete
```

Because the gateway died on every boot, **device approval never completed** — so the
onboarding/pairing screen looped forever and `openclaw doctor` could not be run from it.

## Root cause
While reconfiguring memory to use the bundled **local GGUF embedder** (`llama-cpp`),
the agent edited `openclaw.json` and added an unsupported key:

```json
"llama-cpp": {
  "enabled": true,
  "activation": {        // <-- not a valid plugin-entry key
    "onStartup": true
  }
}
```

`activation` is not part of the plugin-entry schema, so config validation failed. A config
that fails validation blocks gateway startup entirely → exit code 1 on every boot.

This was **not** a file-permission problem (cf. 2026-06-15). The file was owned `1000:1000`
and readable by the `node` user; the container's startup `chown` had already self-healed perms.

### Why the agent's own restarts didn't recover it
- The agent issued `gateway.restart` (process reload), which re-reads config but **cannot get
  past a config that fails validation**.
- Only a config fix — not a restart — resolves a validation failure.
- The crash also took down the Slack channel, so the agent could not act on its own advice
  ("run `openclaw doctor` in a real terminal") from inside Slack.

## Fix
OpenClaw keeps a `openclaw.json.last-good` snapshot. The diff between it and the broken file
was *only* the bad `activation` block (plus the `lastTouchedAt` timestamp) — `llama-cpp` and
`memory-core` were already enabled in `last-good`, so restoring it preserved the embedder fix.

```bash
# copy INSIDE the container as the node user so ownership stays 1000:1000
docker exec -u node openclaw-lynj-openclaw-1 \
  cp /data/.openclaw/openclaw.json /data/.openclaw/openclaw.json.broken-activation
docker exec -u node openclaw-lynj-openclaw-1 \
  cp /data/.openclaw/openclaw.json.last-good /data/.openclaw/openclaw.json
docker exec -u node openclaw-lynj-openclaw-1 openclaw config validate   # -> Config valid
docker restart openclaw-lynj-openclaw-1
```

Gateway came back healthy: `[gateway] ready`, all 9 plugins loaded (incl. `llama-cpp`,
`memory-core`, `slack`). Memory index was then rebuilt with `openclaw memory index --force`.

The broken config is preserved at `/data/.openclaw/openclaw.json.broken-activation`.

## Prevention
- **`activation.onStartup` is not a supported plugin-entry key.** Keep `llama-cpp` as plain
  `"enabled": true`. Whatever "activate on startup" was meant to express isn't a real option.
- **Validate before relying on a restart:** `openclaw config validate` (or `openclaw doctor --lint`)
  catches schema errors *before* they crash-loop the gateway. A `gateway.restart` will not
  surface or survive an invalid config.
- **`openclaw.json.last-good` is the fast revert.** Diff it against the live file first to confirm
  the only delta is the bad change, then restore it (as uid 1000) and do a full container restart.
- Diagnose from a **plain VPS shell** (SSH or Hostinger Browser terminal), not the OpenClaw
  onboarding screen — that screen is a dead end while the gateway is failing to start.

## Follow-up / related risk
The agent runs with `tools.profile: full`, `commands.bash: true`, `tools.elevated.enabled: true`,
and can rewrite its own `openclaw.json` — i.e. it can break itself, as it did here. Combined with
the standing note that anyone who can post in `#crm-openclaw` gets shell on this VPS, the channel
allowlist + tool profile are worth a dedicated hardening pass. See README "Security notes."
