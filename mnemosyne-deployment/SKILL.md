---
name: mnemosyne-deployment
description: "Deploy Mnemosyne: sync server, Hermes provider, DeltaSync."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux]
metadata:
  hermes:
    tags: [mnemosyne, memory, sync, deployment, hermes, coolify]
---

# Mnemosyne Deployment

Deploy the full Mnemosyne memory infrastructure: a central sync server (relay), local Hermes provider integration, and bidirectional DeltaSync — with a dedicated shared-surface database as the agent-crossing memory channel. Full architecture and step-by-step guide.

## Trigger

- "Richte Mnemosyne als Memory ein"
- "Deploy Mnemosyne sync server"
- "Set up agentenübergreifendes Memory"
- "Mnemosyne konfigurieren / administrieren / Troubleshooting / Konsolidierung / Auto-Sleep"
- Any request involving `mnemosyne-oss/mnemosyne` + Hermes + Coolify/Docker

## References

| Datei | Inhalt |
|---|---|
| `references/coolify-sync-service.md` | Coolify-API-Deploy-Recipe für den Sync-Server |
| `references/sync-cron-script.md` | no_agent-Watchdog-Cron für bidirektionalen Sync |
| `references/operations.md` | **Provider-Betrieb**: Auto-Sleep/Konsolidierung, Versions-Bugs (#506–#537), Profil-Isolation, Junk-Cleanup, Diagnose (integriert aus AtlasOmnia/hermes-custom-pack) |

## Architecture (Conceptual)

```
┌──────────────────────────────────┐
│  Zentrale Sync-Instanz (Coolify) │  ← sync-serve (Port 8765, API-Key)
│  relay.db (dediziertes Volume)    │     DeltaSync HTTP: /sync/pull, /sync/push
└──────────┬───────────────────────┘
           │  HTTPS (Traefik/TLS)
     ┌─────┴──────┐
     │            │
┌────▼────────┐ ┌─▼──────────────┐
│ Hermes      │ │ Weitere Agents │  ← je eigener Container, eigene SQLite
│ Provider-DB │ │ (später)       │
│ + Surface   │ │ + Surface      │
└─────────────┘ └────────────────┘
```

**Key rules** (from upstream docs):
- Central SQLite DB is NEVER shared as a filesystem between containers
- Sync uses a **dedicated shared-surface DB** — never point `--db-path` at the private BEAM/profile DB
- Sync is event-sourced (DeltaSync), deduplicated by `event_id`, bidirectional
- Optionally encrypted client-side (`--encrypt` / `--encrypt-key-file`)

## Prerequisites

- Hermes Agent running (container or bare-metal)
- Persistent volume for `~/.hermes/` (survives container rebuilds)
- Coolify host (or any Docker host) for the central sync server
- Domain for the sync server (e.g., `mnemosyne.followthegreensheep.de`)

## Step 1 — Local Instance (Hermes Container)

The provider runs as a side-venv on the persistent Hermes volume (survives rebuilds).

### 1a. Create Side-Venv

Use the SAME Python major.minor as the Hermes gateway:

```bash
# Find gateway Python
/app/.venv/bin/python --version  # e.g. Python 3.11.15

# Create venv on persistent volume
/app/.venv/bin/python -m venv /root/.hermes/venvs/mnemosyne
```

### 1b. Install mnemosyne-memory + Hermes plugin

```bash
/root/.hermes/venvs/mnemosyne/bin/pip install --upgrade 'mnemosyne-memory[embeddings]' mnemosyne-hermes pynacl
```

`[embeddings]` is required for the wrapper install — fastembed/onnxruntime on ARM64 is supported.
**`pynacl` MUST be installed separately** — the `[sync]` extra does NOT include it
(confirmed Aug 2026). Without it, any `--encrypt`/`--encrypt-key-file` sync dies with
`ModuleNotFoundError: No module named 'nacl'`.

### 1c. Install Hermes Plugin (Wrapper Mode)

```bash
export HERMES_HOME=/root/.hermes
export MNEMOSYNE_DATA_DIR=/root/.hermes/mnemosyne
/root/.hermes/venvs/mnemosyne/bin/mnemosyne-hermes install \
  --mode wrapper \
  --python /root/.hermes/venvs/mnemosyne/bin/python
```

This creates `/root/.hermes/plugins/mnemosyne/` and installs the override skill at
`/root/.hermes/skills/memory/mnemosyne-memory-override/SKILL.md`.

### 1d. Activate Provider

```bash
hermes config set memory.provider mnemosyne
# Surface-Pfad auf die SYNC-DB zeigen lassen — sonst schreiben die
# Provider-Tools (mnemosyne_shared_*) in eine ANDERE DB als die, die synct!
hermes config set memory.mnemosyne.shared_surface_path /root/.hermes/mnemosyne/shared-surface.db
hermes config set memory.mnemosyne.shared_surface_read true
```

⚠️ **CRITICAL NAMESPACE**: The provider reads Hermes config keys from
`memory.mnemosyne.<key>`, NOT `memory.<key>`. `hermes config set memory.shared_surface_path`
is saved to config.yaml but **silently ignored** by the provider (falls back to the
default `~/.mnemosyne/data/shared/mnemosyne.db`). Verify with `mnemosyne_shared_stats`:
it reports the `shared_db` path the provider actually uses.

Verify:

```bash
hermes memory status
# Must show: Provider: mnemosyne, Status: available ✓
```

### 1e. Persistence: point the provider at the persistent volume (config, NOT symlink)

The provider defaults its data dir to `~/.mnemosyne/` — which is NOT on the
persistent volume in the Hermes gateway container (only `/root/.hermes` is).

**DO NOT use a symlink** (`ln -s /root/.hermes/mnemosyne /root/.mnemosyne`):
symlinks live in the container's writable layer and are **destroyed on every
container restart** — the provider then silently recreates `/root/.mnemosyne`
as a fresh directory on the ephemeral layer and ALL memories are lost.

The durable fix is the Mnemosyne config file, which the provider reads at startup:

```bash
# The Mnemosyne config at /root/.hermes/mnemosyne/config.yaml (created by the
# wrapper install) already sets: data_dir: /root/.hermes/mnemosyne
# That key redirects the private BEAM-DB to the persistent volume.

# Set it explicitly if missing:
/root/.hermes/venvs/mnemosyne/bin/mnemosyne config set data_dir /root/.hermes/mnemosyne

# The SHARED surface DB is redirected via the Hermes config key (step 1d):
# memory.mnemosyne.shared_surface_path = /root/.hermes/mnemosyne/shared-surface.db
```

After a restart, verify with `mnemosyne_shared_stats` (tool) that `shared_db`
points at `/root/.hermes/mnemosyne/...` — NOT `/root/.mnemosyne/...`.

### 1f. Restart Gateway

The Gateway must restart for the provider to take effect. In Docker/Coolify:
restart the Hermes container via its management tooling (Coolify UI → Restart,
or `POST /api/v1/applications/{uuid}/restart`).

After restart, re-verify: `hermes memory status`.

## Step 2 — Central Sync Server (Coolify)

Deploy `mnemosyne sync-serve` as a Docker Compose service with its own volume.

See `references/coolify-sync-service.md` for the full Coolify API recipe with copy-paste deploy script.

Core compose template (⚠️ corrected Aug 2026 — the naive `command: >` with backslash
continuations causes a CRASH LOOP; Coolify double-escapes `\\` into `\\\\n`):

```yaml
services:
  mnemosyne-sync:
    image: python:3.12-slim       # multi-arch (AMD64/ARM64)
    restart: unless-stopped
    expose:
      - "8765"
    volumes:
      - mnemosyne-sync-data:/data
    environment:
      - MNEMOSYNE_DATA_DIR=/data
      - MNEMOSYNE_SYNC_API_KEY=<generated-key>
    command:
      - sh
      - -ec
      - |
        echo "$$MNEMOSYNE_SYNC_API_KEY" > /data/api.key
        chmod 600 /data/api.key
        pip install --quiet 'mnemosyne-memory[embeddings,sync]'
        exec mnemosyne sync-serve --host 0.0.0.0 --port 8765 --db-path /data/relay.db --initialize-surface --behind-tls-proxy --api-key-file /data/api.key
    healthcheck:
      test: ["CMD", "python3", "-c", "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8765/healthz', timeout=5)"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 120s
volumes:
  mnemosyne-sync-data:
```

Three non-negotiable flags (all discovered the hard way):

1. **`--behind-tls-proxy`** — without it the server REFUSES to bind non-loopback:
   `ValueError: HTTPS is required beyond loopback unless behind_tls_proxy is explicitly enabled`.
   Traefik terminates TLS, so this flag is required on Coolify.
2. **`--api-key-file /data/api.key`** — write the env var to a file first, then use
   the file flag. Keeps the key out of process args and avoids `$`-interpolation hell.
3. **No backslash line continuations** — use the block-scalar `|` pattern above with
   ONE shell statement per line. Also note `$$ENV_VAR` (double dollar) is required
   for env interpolation inside Coolify-rendered compose.

**ARM64**: `python:3.12-slim` is multi-arch ✓. `pip install` (fastembed/onnxruntime) takes ~2-4 minutes on first start.

### Domain Assignment (UI-only for Compose Services)

Coolify API v4.1.x rejects `docker_compose_domains` on POST/PATCH (422). Domain must be assigned in the Coolify UI:

1. Coolify → Project "Mnemosyne" → Service "mnemosyne-sync"
2. Sub-application → Domain → Add `https://mnemosyne.followthegreensheep.de`
3. Save → Redeploy

## Step 3 — Sync Setup

### 3a. Initialize Shared-Surface DB (local)

```bash
export PATH=/root/.hermes/venvs/mnemosyne/bin:$PATH
export MNEMOSYNE_DATA_DIR=/root/.hermes/mnemosyne

mnemosyne sync-init --db-path /root/.hermes/mnemosyne/shared-surface.db
```

Output: `{"status": "initialized", "session_id": "hermes_shared_surface"}`.

### 3b. Save Credentials

```bash
cp /tmp/mnemosyne-sync-api.key /root/.hermes/mnemosyne/sync-api.key
cp /tmp/mnemosyne-sync-encryption.key /root/.hermes/mnemosyne/sync-encryption.key
chmod 600 /root/.hermes/mnemosyne/sync-*.key
```

### 3c. First Sync Test — Cloudflare WAF workaround REQUIRED

The mnemosyne client calls the remote via `urllib.request.urlopen` internally,
which sends `User-Agent: Python-urllib/x.y`. Coolify instances behind Cloudflare
BLOCK that with `HTTP 403 / error code: 1010`:

```
Errors (1): HTTP 403 on /sync/push: error code: 1010
```

**Fix**: install a global urllib opener with a browser UA BEFORE importing the CLI.
This cannot be done via the CLI flags — it must be a wrapper script:

```python
#!/root/.hermes/venvs/mnemosyne/bin/python
import os, sys, urllib.request
UA = ("Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 "
      "(KHTML, like Gecko) Chrome/126.0.0.0 Safari/537.36")
_opener = urllib.request.build_opener()
_opener.addheaders = [("User-Agent", UA), ("Accept", "application/json")]
urllib.request.install_opener(_opener)

from mnemosyne.cli import run_cli  # import AFTER opener install
sys.argv = [
    "mnemosyne", "sync",
    "--db-path", "/root/.hermes/mnemosyne/shared-surface.db",
    "--remote", "https://mnemosyne.followthegreensheep.de",
    "--api-key-file", "/root/.hermes/mnemosyne/keys/sync-api.key",
    "--mode", "bidirectional",
    "--encrypt-key-file", "/root/.hermes/mnemosyne/keys/sync-encryption.key",
]
run_cli()
```

Note: `--encrypt-key-file` (NOT `--encrypt VALUE`) — `--encrypt` expects the key
as an argument and errors with `argument --encrypt: expected one argument`.

### 3d. Sync Cron (Hermes Cron Job)

See `references/sync-cron-script.md` for the `--no-agent true` watchdog pattern.
Run every 15 min; script exits 0 silently on success, prints output + exits 1 on
failure (watchdog: non-empty stdout = alert).

## Step 4 — Verification

### Provider Check

```bash
hermes memory status
# Must show: Provider: mnemosyne, Status: available ✓
```

### Sync Check

```bash
export PATH=/root/.hermes/venvs/mnemosyne/bin:$PATH
mnemosyne sync-status --db-path /root/.hermes/mnemosyne/shared-surface.db \
  --remote https://mnemosyne.followthegreensheep.de \
  --api-key-file /root/.hermes/mnemosyne/sync-api.key --json
# Expect: Connected: true; after first push: "Synced events: >0"
```

NOTE: `sync-status` REQUIRES `--db-path` (errors with
`the following arguments are required: --db-path` if omitted).

### Database Files

```bash
ls -la /root/.hermes/mnemosyne/data/mnemosyne.db   # BEAM-DB (private)
ls -la /root/.hermes/mnemosyne/shared-surface.db    # Shared-Surface DB (synced)
```

## Step 5 — Future: Bind Additional Agents

### MCP Server

```bash
/root/.hermes/venvs/mnemosyne/bin/mnemosyne mcp --transport sse --port 8080
```

### Python SDK

```python
from mnemosyne import Mnemosyne
mem = Mnemosyne(db_path="/path/to/shared-surface.db", session_id="agent-xyz")
mem.remember("User prefers dark mode", scope="global")
results = mem.recall("dark mode", limit=5)
# Then run mnemosyne sync against this DB
```

## Pitfalls

### Provider reads config keys ONLY at startup — changes need a Gateway restart

`shared_surface_path` (and other `memory.mnemosyne.*` keys) are read once when
the provider initializes. Setting the key while the Gateway runs has NO effect
on the running provider — it keeps using the old path. After changing any
`memory.mnemosyne.*` key, restart the Hermes Gateway, then verify with
`mnemosyne_shared_stats` (reports the `shared_db` path in use).

### Config namespace: `memory.mnemosyne.<key>`, NOT `memory.<key>`

The provider's `_read_config_key()` looks in Hermes config.yaml under
`memory.mnemosyne.<key>` (and falls back to the Mnemosyne config singleton).
`hermes config set memory.shared_surface_path ...` succeeds but is silently
ignored. Always verify the running path via `mnemosyne_shared_stats` after setup.

### Container restarts destroy symlinks — never symlink for persistence

A symlink created inside a container (`ln -s /root/.hermes/mnemosyne /root/.mnemosyne`)
lives in the ephemeral writable layer. Any container restart (Coolify Restart,
rebuild, redeploy) removes it, and software that resolved data through it falls
back to creating fresh data on the ephemeral layer — silent data loss. Always
persist via config keys / env vars / volumes that live on the persistent volume.

### Do NOT sync the private BEAM-DB

Upstream docs: "Do not point sync at a private profile/session database."
Always use a **dedicated** shared-surface DB (created via `sync-init`).
Private BEAM-DB: `~/.hermes/mnemosyne/data/mnemosyne.db`.
Sync DB: `~/.hermes/mnemosyne/shared-surface.db`.

### Domain assignment is UI-only for Compose Services

Coolify v4.1.x cannot assign domains to service sub-apps via API.
`docker_compose_domains` is rejected with 422 on both POST and PATCH.
See `references/coolify-sync-service.md` for the API sequence and UI fallback.

### `exited` status after creation is normal

The `python:3.12-slim` image has no Mnemosyne pre-installed. First start runs
`pip install mnemosyne-memory[embeddings,sync]` (2-4 min) then sync-serve.
Expect the deploy to take a few minutes.

### Venv must be on persistent volume

`/root/.hermes/venvs/mnemosyne` MUST be on the Hermes persistent data volume.
If on the container filesystem (`/app/`), it is lost on every rebuild.

### Python major.minor must match Gateway

Side-venv Python must match the Hermes Gateway interpreter's major.minor.
Check: `/app/.venv/bin/python --version`. Mismatch causes import errors.

### Coolify volume persistence

Use explicit `volumes:` in Docker Compose YAML (not `custom_docker_run_options -v`).
See `coolify-api` skill: "custom_docker_run_options volume mounts are NOT persistent".

### Crash loop / `restarting:unknown` on the sync service

If the sync container loops (`restarting:unknown`, `degraded:unhealthy`, healthz 503),
pull the Coolify UI container logs FIRST — the traceback names the exact cause.
Diagnosed causes (in order of likelihood for a fresh deploy):

1. **Backslash corruption** — compose `command:` with `\\` continuations renders as
   `\\\\n` → shell breaks. Re-deploy with the block-scalar pattern above.
2. **Missing `--behind-tls-proxy`** — `ValueError: HTTPS is required beyond loopback...`
3. **`pip install` still running** on first boot (2-4 min on ARM64) — wait before
   assuming failure. The log shows the pip notice, then the server traceback.

### Surface ID must match between client and server

If push returns `"event belongs to a different or unscoped surface"`, the event
lacks the expected `surface_id`. Both `sync-init` (local) and `--initialize-surface`
(server) default to `surface_id: "shared-surface-v1"` — client events carry it
automatically from the surface DB. A manual test event without `surface_id` fails
with exactly this error; add `"surface_id": "shared-surface-v1"` when hand-testing.

### Server enforces client-side encryption

A plaintext push is rejected with `"plaintext payload rejected because encryption
is required"`. The relay is a BLIND relay — always sync with `--encrypt-key-file`.
All agents sharing a surface must use the SAME encryption key file.
