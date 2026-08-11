# Coolify Sync Service — API Deployment Recipe

Deploy the Mnemosyne sync server (`sync-serve`) as a Docker Compose service on Coolify via its REST API.

## Prerequisites

- Coolify API key stored in `/tmp/coolify_key.txt` (format: `4|...`)
- Project "Mnemosyne" with a `production` environment
- Server UUID: `yogg48ww4k84oo4w8kc0gckg` (localhost)
- Generated API key at `/tmp/mnemosyne-sync-api.key`
- Generated encryption key at `/tmp/mnemosyne-sync-encryption.key`

## Deploy Script

Save as `deploy_mnemosyne_sync.py` and run with `/app/.venv/bin/python`:

```python
import base64, json, os, urllib.request

with open("/tmp/coolify_key.txt") as f:
    raw = f.read().strip()
parts = raw.split("|", 1)
token = "|".join(parts)

def api(method, path, body=None):
    req = urllib.request.Request(
        f"https://coolify.followthegreensheep.de{path}",
        method=method,
        headers={
            "Authorization": "Bearer " + token,
            "User-Agent": "Mozilla/5.0",
            "Accept": "application/json",
            "Content-Type": "application/json",
        },
        data=json.dumps(body).encode() if body else None,
    )
    try:
        with urllib.request.urlopen(req, timeout=30) as resp:
            return resp.status, json.loads(resp.read().decode() or "{}")
    except urllib.error.HTTPError as e:
        return e.code, json.loads(e.read().decode() or "{}")

# Read API key from file (never a literal in source — masking risk)
with open("/tmp/mnemosyne-sync-api.key") as f:
    api_key = f.read().strip()

# Build compose YAML with string concatenation (avoids credential masking)
# ⚠️ CORRECTED Aug 2026: block-scalar command, NO backslash continuations,
# --behind-tls-proxy required (Traefik terminates TLS), --api-key-file.
lines = []
lines.append("services:")
lines.append("  mnemosyne-sync:")
lines.append("    image: python:3.12-slim")
lines.append("    restart: unless-stopped")
lines.append("    expose:")
lines.append('      - "8765"')
lines.append("    volumes:")
lines.append("      - mnemosyne-sync-data:/data")
lines.append("    environment:")
lines.append("      - MNEMOSYNE_DATA_DIR=/data")
lines.append("      - MNEMOSYNE_SYNC_API_KEY=" + api_key)
lines.append("    command:")
lines.append("      - sh")
lines.append("      - -ec")
lines.append("      - |")
lines.append('        echo "$$MNEMOSYNE_SYNC_API_KEY" > /data/api.key')
lines.append("        chmod 600 /data/api.key")
lines.append("        pip install --quiet 'mnemosyne-memory[embeddings,sync]'")
lines.append("        exec mnemosyne sync-serve --host 0.0.0.0 --port 8765 --db-path /data/relay.db --initialize-surface --behind-tls-proxy --api-key-file /data/api.key")
lines.append("    healthcheck:")
lines.append('      test: ["CMD", "python3", "-c", "import urllib.request; urllib.request.urlopen(\'http://127.0.0.1:8765/healthz\', timeout=5)"]')
lines.append("      interval: 30s")
lines.append("      timeout: 10s")
lines.append("      retries: 5")
lines.append("      start_period: 120s")
lines.append("volumes:")
lines.append("  mnemosyne-sync-data:")
compose_yaml = "\n".join(lines) + "\n"

# Create service
b64 = base64.b64encode(compose_yaml.encode()).decode()
status, resp = api("POST", "/api/v1/services", {
    "name": "mnemosyne-sync",
    "project_uuid": "r56a1ue1stwuu69w15jrf5f2",
    "server_uuid": "yogg48ww4k84oo4w8kc0gckg",
    "environment_uuid": "<YOUR_ENV_UUID>",
    "docker_compose_raw": b64,
})
print(f"Service-Create: HTTP {status}")
print(json.dumps(resp, indent=1)[:500])

if status in (200, 201):
    svc_uuid = resp.get("uuid", "")
    print("\nService-UUID:", svc_uuid)
    # Deploy
    st2, resp2 = api("POST", f"/api/v1/deploy?uuid={svc_uuid}")
    print(f"\nDeploy: HTTP {st2}")
    print(json.dumps(resp2, indent=1)[:400])
```

## Domain Assignment (UI-only)

After deployment, the domain MUST be assigned in the Coolify UI:

1. Open Coolify → Project "Mnemosyne"
2. Service "mnemosyne-sync" → sub-application
3. Domain tab → Add `https://mnemosyne.followthegreensheep.de`
4. Save → Redeploy

The Coolify API v4.1.x rejects `docker_compose_domains` on POST/PATCH (422)
for Compose services. This is Coolify Issue #9502.

## Post-Install Verification

```bash
# Check /healthz on the sync server
curl -s https://mnemosyne.followthegreensheep.de/healthz
# Expect: {"status": "ok"}

# Check service status via API
KEY=$(cat /tmp/coolify_key.txt)
curl -s -H "Authorization: Bearer $KEY" -H "User-Agent: Mozilla/5.0" \
  "https://coolify.followthegreensheep.de/api/v1/services" \
  | python3 -c "import sys,json; [print(s.get('name'),s.get('status')) for s in json.load(sys.stdin) if 'mnemosyne' in s.get('name','')]"
# Expect: mnemosyne-sync running:healthy (or running:unknown — HEALTHCHECK may be unregistered)
```

## Troubleshooting

### Service shows `exited` after creation

Normal — `python:3.12-slim` has no Mnemosyne pre-installed. The container runs
`pip install` on first start (2-4 minutes on ARM64). Wait and redeploy if needed.

### Crash loop / `restarting:unknown` / healthz 503

Pull container logs from the Coolify UI FIRST — the traceback names the cause.
Known causes, in order of likelihood:

1. **Backslash corruption** (compose `command: >` with `\` continuations renders
   as `\\\\n` → shell breaks). Use the block-scalar pattern above. Diagnosis: the
   rendered `docker_compose` in the API shows `\\\\n` sequences.
2. **Missing `--behind-tls-proxy`**:
   `ValueError: HTTPS is required beyond loopback unless behind_tls_proxy is explicitly enabled`
   — add the flag; Traefik terminates TLS in front of the container.
3. **`pip install` still running** on first boot — wait 2-4 min before assuming failure.

After PATCHing the compose, deploy again via `POST /api/v1/deploy?uuid={svc_uuid}`.
The API deployments list stays empty for Compose services (normal — see coolify-api
skill); use service status + healthz to verify.

### `running:unknown` but serving traffic

`running:unknown` is common for Docker Image and Compose Service apps that lack
a Docker HEALTHCHECK recognized by Coolify. The `/healthz` endpoint still works.
Check with an external `curl`.

### ARM64 compatibility

- `python:3.12-slim` — multi-arch ✓
- `onnxruntime` (via fastembed) — aarch64 wheels exist ✓
- If fastembed fails on ARM64, consider using `mnemosyne-memory[sync]` only
  (no embeddings) for the relay server. The sync-serve doesn't perform vector
  search itself — it only relays events.
