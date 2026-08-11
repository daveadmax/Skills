# Sync Cron Script (`no_agent` Watchdog Pattern) — TESTED Aug 2026

Periodic bidirectional Mnemosyne DeltaSync without LLM overhead.
**Must be a Python wrapper** — see the Cloudflare WAF note below; a plain bash
call to the `mnemosyne` CLI dies with `HTTP 403 / error code: 1010`.

## Script

Save as `~/.hermes/scripts/mnemosyne-sync-job.py` (shebang points at the side-venv):

```python
#!/root/.hermes/venvs/mnemosyne/bin/python
"""Mnemosyne-Sync-Job (Watchdog-Muster für Cron).

- Erfolg: keine Ausgabe, Exit 0 (Cron bleibt still)
- Fehler: Sync-Ausgabe + Exit 1 (Cron sendet Fehler-Alert)

Hinweis: Der Cron-Scheduler führt .py-Skripte mit SEINER eigenen Python aus
(Shebang wird ignoriert). Deshalb Re-Exec-Bootstrap: immer in die
Mnemosyne-Venv-Python wechseln, sonst ModuleNotFoundError.
"""
import os
import sys

if "/root/.hermes/venvs/mnemosyne" not in sys.executable:
    # WICHTIG: argv[0] = voller Pfad, sonst wird sys.executable zu
    # /usr/local/bin/python aufgelöst → Endlos-Re-Exec-Loop (95% CPU)!
    _script = os.path.abspath(__file__)
    os.execv(
        "/root/.hermes/venvs/mnemosyne/bin/python",
        ["/root/.hermes/venvs/mnemosyne/bin/python", _script] + sys.argv[1:],
    )

import contextlib
import io
import os
import sys
import urllib.request

# Cloudflare-WAF-Workaround: globaler Opener mit Browser-UA VOR dem CLI-Import.
# Der mnemosyne-Client nutzt intern urllib.request.urlopen → ohne Browser-UA
# blockt Cloudflare mit 403 error code 1010.
UA = (
    "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 "
    "(KHTML, like Gecko) Chrome/126.0.0.0 Safari/537.36"
)
_opener = urllib.request.build_opener()
_opener.addheaders = [("User-Agent", UA), ("Accept", "application/json")]
urllib.request.install_opener(_opener)

from mnemosyne.cli import run_cli  # noqa: E402

KEYS_DIR = "/root/.hermes/mnemosyne/keys"
DB = "/root/.hermes/mnemosyne/shared-surface.db"
REMOTE = os.environ.get("MNEMOSYNE_SYNC_REMOTE", "https://mnemosyne.followthegreensheep.de")

sys.argv = [
    "mnemosyne", "sync",
    "--db-path", DB,
    "--remote", REMOTE,
    "--api-key-file", os.path.join(KEYS_DIR, "sync-api.key"),
    "--mode", "bidirectional",
    "--encrypt-key-file", os.path.join(KEYS_DIR, "sync-encryption.key"),
]

buf = io.StringIO()
try:
    with contextlib.redirect_stdout(buf):
        run_cli()
    out = buf.getvalue()
    if "Errors" in out and "Errors (0)" not in out:
        print("Mnemosyne-Sync FEHLER:")
        print(out)
        sys.exit(1)
    # Erfolg: still (Watchdog-Pattern)
    sys.exit(0)
except SystemExit as e:
    out = buf.getvalue()
    if e.code not in (0, None):
        print("Mnemosyne-Sync FEHLER (Exit %s):" % e.code)
        print(out)
        sys.exit(e.code if isinstance(e.code, int) else 1)
    sys.exit(0)
except Exception as e:
    print("Mnemosyne-Sync EXCEPTION:", e)
    sys.exit(1)
```

## Cron Job Registration

```python
cronjob(
    action="create",
    name="Mnemosyne-Sync",
    schedule="*/15 * * * *",
    no_agent=True,
    script="mnemosyne-sync-job.py",  # relative to ~/.hermes/scripts/
)
```

## Key design choices (all tested)

- **Re-Exec-Bootstrap (oben im Skript)**: Der Cron-Scheduler führt `.py`-Skripte
  mit SEINER eigenen Python aus — das Shebang wird ignoriert. Ohne Bootstrap
  stirbt der Job mit `ModuleNotFoundError: No module named 'mnemosyne'`.
  `argv[0]` MUSS der volle Venv-Pfad sein; `"python"` → `sys.executable` wird
  zu `/usr/local/bin/python` aufgelöst → Endlos-Re-Exec-Loop (95 % CPU).
  Simulierter Cron-Test: `/app/.venv/bin/python mnemosyne-sync-job.py` (darf
  nicht hängen, Exit 0). Details: `cronjob-patterns` Skill, Pitfall
  "Cron scheduler runs `.py` scripts with ITS OWN Python".
- **`no_agent=True`**: No LLM involved — the script IS the job. Non-empty stdout
  is delivered verbatim; EMPTY stdout = silent tick; non-zero exit = error alert.
- **stdout redirect + "Errors" scan**: the sync CLI prints a success summary
  ("Push: Accepted: N…") on every run — without the redirect, the user gets
  spammed each 15 minutes. Only print on real failures.
- **Cloudflare UA patch must precede CLI import**: `run_cli()` builds the client
  at call time, but `urllib` opener must be installed first — importing the CLI
  module doesn't make the network call, but keeping the patch on top is the
  safe order.
- **`--encrypt-key-file`** (not `--encrypt KEY`): the relay is a blind relay;
  plaintext pushes are rejected with "plaintext payload rejected because
  encryption is required".
- Keys live in `~/.hermes/mnemosyne/keys/` (persistent volume, chmod 600) —
  NOT `/tmp/` (wiped on reboot).

## Manual test

```bash
chmod +x /root/.hermes/scripts/mnemosyne-sync-job.py
/root/.hermes/scripts/mnemosyne-sync-job.py; echo "Exit: $?"   # → silent, 0
```
