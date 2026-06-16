# Incident: gateway crash-loop from config file permissions (2026-06-15)

## Symptom
OpenClaw gateway repeatedly exited shortly after start:

```
Failed to read config at /data/.openclaw/openclaw.json Error: EACCES: permission denied
[plugins/runtime] plugin host registry cleanup failed: Error: EACCES ... open '/data/.openclaw/openclaw.json'
WARN: OpenClaw exited with code 0
```

## Root cause
The app process runs as user **`node` (uid 1000)** (`runuser -u node -- node server.mjs`).
`openclaw.json` had been edited from the **host as root**, which left it owned by `root:root` mode `600`. uid 1000 could no longer read it → `EACCES` on every config load / live reload.

(`docker exec` defaults to root, so a `cat` test *passes* and hides the problem — always test as the app user: `docker exec -u node ... cat /data/.openclaw/openclaw.json`.)

## Fix
```bash
chown 1000:1000 /docker/openclaw-lynj/data/.openclaw/openclaw.json \
                /docker/openclaw-lynj/data/.openclaw/openclaw.json.bak
docker restart openclaw-lynj-openclaw-1
```
Gateway came back healthy (`[gateway] ready`, `config valid`).

## Prevention
- **Never hand-edit `openclaw.json` as host root.** Use the CLI (`openclaw config set`, `openclaw channels add`) or edit as uid 1000.
- The container `entrypoint.sh` runs `chown -R node:node /data` on every start, so a **container restart self-heals** this. The crash window only opens when the file is changed (as root) while the container is already running and a live reload tries to read it.
