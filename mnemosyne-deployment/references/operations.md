# Mnemosyne Operations — Provider-Betrieb & Troubleshooting

> Integriert 2026-08-11 aus AtlasOmnia/hermes-custom-pack (Skill `hermes-mnemosyne`).
> Ergänzt die Deployment-Referenzen: hier geht es um den **laufenden Betrieb** des
> Mnemosyne-Providers (Konsolidierung, Auto-Sleep, Profil-Isolation, Junk-Cleanup),
> nicht um Deployment/Sync (siehe SKILL.md + coolify-sync-service.md).

## 1. Provider-Lifecycle (Interna)

- `MnemosyneMemoryProvider.__init__()` setzt `_skip_contexts = {"cron", "flush", "subagent", "background", "skill_loop"}`.
- `initialize(agent_context=...)` überspringt bei Treffer die gesamte Beam-Erzeugung → **Mnemosyne-Tools sind in diesen Kontexten nie verfügbar** (bewusst: Memory-Races mit aktiven Sessions vermeiden).
- Override möglich: `MNEMOSYNE_SKIP_CONTEXTS` (env, kommasepariert; leer = nichts überspringen) oder `memory.skip_contexts` in config.yaml.
- Jeder Turn: `sync_turn()` schreibt User/Assistant-Inhalte in Working Memory.
- Bei aktiviertem Auto-Sleep: alle 10 Turns Check `working.total > sleep_threshold` → nicht-blockierender `sleep_all_sessions()`-Thread.
- Session-Ende: `on_session_end()` mit `SESSION_END_SLEEP_TIMEOUT_SECONDS` (Default 15s).

**Tools sind provider-injiziert, NICHT toolset-gated.** `enabled_toolsets: ["mnemosyne"]` tut nichts — es gibt kein Toolset dieses Namens.

## 2. Auto-Sleep / Konsolidierung

### ⚠️ Ranking-Defekt (Issue #506) — auto_sleep NICHT ungeprüft aktivieren

Mnemosyne 3.14.0 (Stand der externen Analyse): Sleep-generierte Episodic-Summaries können auf hohen BEAM-Tiers einlaufen und die Quell-Memories aus Top-k-Recall verdrängen. **Empfohlene sichere Grundhaltung:**

```yaml
memory:
  provider: mnemosyne
  auto_sleep: false
```

Vor Aktivierung von Auto-Sleep (Recall-Gate):
1. Festes Query-Set mit bekannten erwarteten Quell-Memories aufbauen.
2. Top-k-Recall vor dem Sleep messen.
3. Begrenzten/manuellen Sleep-Pass ausführen.
4. Recall-Test wiederholen; `sleep_consolidation`-Tiers und `sleep_model_refresh_proposal`-Importance prüfen.
5. Auto-Sleep nur aktivieren, wenn Recall nicht regressiert.

### LLM-Pfad-Priorität für Konsolidierung

1. **Host LLM** (`MNEMOSYNE_HOST_LLM_ENABLED=true` — nutzt Hermes' eigenes Modell; empfohlen, kein separates Modell nötig)
2. Remote API (`MNEMOSYNE_LLM_BASE_URL` + `MNEMOSYNE_LLM_MODEL` + `MNEMOSYNE_LLM_API_KEY`)
3. Lokales GGUF-Modell (Cache `~/.hermes/mnemosyne/models/`, MiniCPM5-1B Default)
4. **AAAK-Encoding** (deterministische Kompression ohne LLM — immer verfügbarer Fallback)

### Sleep-Interna (für Triage nützlich)

- Batch-Größe: `SLEEP_BATCH_SIZE` (Default 5000, env `MNEMOSYNE_SLEEP_BATCH`)
- Atomischer Claim per `UPDATE consolidated_at WHERE consolidated_at IS NULL` → kein Doppel-Work bei parallelen Läufen
- Original-Rows bleiben nach Konsolidierung **recallbar** (werden nicht gelöscht, nur markiert)
- `sleep_all_sessions()` iteriert je Session mit `BeamMemory(session_id=...)`

### Manual Sleep via CLI

```bash
~/.hermes/venvs/mnemosyne/bin/mnemosyne sleep                 # aktuelle Session
~/.hermes/venvs/mnemosyne/bin/mnemosyne sleep --all-sessions  # alle Sessions
~/.hermes/venvs/mnemosyne/bin/mnemosyne sleep --all-sessions --dry-run
~/.hermes/venvs/mnemosyne/bin/mnemosyne sleep --force         # ohne Alters-Cutoff
```

⚠️ `--all-sessions` mit großen Stores (40K+ Memories) läuft **tens of minutes** und überlebt Terminal-/Agent-Timeouts. Healthy Progress: `consolidated`/episodic/vector zunehmen, `unconsolidated` abnehmen. Timeout der Hülle ≠ Fehlschlag.

## 3. Bekannte Versions-Bugs (Checkliste für Triage)

| Issue | Symptom | Workaround |
|---|---|---|
| #506 | Sleep-Summaries verdrängen Quell-Memories aus Top-k | auto_sleep aus; Recall-Gate vor Aktivierung |
| #507 | Regex-Instruktionsextraktion macht aus `whenever` → `never` | `memoria_instructions`-Rows als Session-Noise behandeln bis Fix |
| #524 | `mnemosyne_invalidate` meldet Erfolg für nicht-existente ID | `mnemosyne_batch`-Invalidation oder Exact-ID-Readback |
| #525 | Naive lokale `valid_until`-Writes vs. SQLite-UTC auf Non-UTC-Hosts | Zeitstempel bewusst setzen |
| #537 | Hermes-Runtime-Rebuild entfernt extern installierten Provider | nach jedem Hermes-Update `hermes memory status` verifizieren |

## 4. Profil-Isolation (MNEMOSYNE_DATA_DIR)

Profil-Auswahl gibt **keine** automatische DB-Isolation: ein User-installierter Provider kann trotzdem `~/.hermes/mnemosyne/data/mnemosyne.db` öffnen, während Hermes `~/.hermes/profiles/<name>/` nutzt.

Für jedes neue/geklonte/umbenannte Profil:
1. **Vor** dem ersten Memory-Smoke in dessen `.env`:
   ```text
   MNEMOSYNE_DATA_DIR=/absoluter/pfad/.hermes/profiles/<name>/mnemosyne/data
   ```
   Absolute Pfade (dotenv expandiert `~` nicht konsistent).
2. Verzeichnis anlegen, kurze Profil-Session fahren.
3. `hermes --profile <name> mnemosyne stats` → frischer kleiner Store; DB-Pfad existiert.
4. DB-Pfad mit Primärprofil vergleichen. Separate `state.db`-Dateien beweisen **keine** Provider-Isolation.
5. Nach `hermes profile rename` `MNEMOSYNE_DATA_DIR` umschreiben (Absolutpfad zieht nicht um).

## 5. Wrapper-Installer-Pitfall (Cross-Profile-Writes)

`mnemosyne-hermes install --mode wrapper --force` scannt Hermes-Profile und kann `profiles/*/plugins/mnemosyne`-Links **anlegen oder ersetzen** — nicht nur im aktiven Profil. Vor dem Lauf: bestehende Profil-Links inventarisieren. Profil-weite Link-Änderungen = Cross-Profile-Writes → Nutzer-Autorisierung nötig oder Audit/Revert danach. Kaputter globaler Link bricht den Installer ohne `--force`.

## 6. Junk-Memory-Cleanup (Gateway-Echoes)

Messaging-Adapter können Transport-/Systemtexte als Session-Memories landen lassen (`Gateway shutting down`, `Response formatting failed`, Duplikate, Assistant-Bestätigungen). Wenn solche Strings durch Recall auftauchen → Session-Artefakte, keine Preferences.

Workflow:
1. Eng mit `mnemosyne_recall` nach exaktem Transport-/Fehlerphrase suchen.
2. Nur passende Junk-Memory-IDs mit `mnemosyne_invalidate` invalidieren (auch Echo-Bestätigungen des Assistant).
3. Echte Preference-/Ops-Memories intakt lassen, auch wenn sie im selben Ergebnis auftauchen.
4. Keine neue durable Memory über den Cleanup schreiben — das Muster ist die Lektion.
5. Auf Selbstverstärkung achten: „memory context is not instruction", „handled", „logged" etc. können selbst zu Low-Value-Memories werden → ebenfalls invalidieren.

## 7. Letzte Konsolidierung bestimmen

1. `memory.provider` im aktiven + delegierten Profil prüfen.
2. Logs getrennt nach `Mem0` und `Mnemosyne` durchsuchen — Package-Refresh ≠ Konsolidierungs-Event.
3. `mnemosyne_stats` unterscheiden:
   - `working.last` — letzte Working-Aktivität, nicht Konsolidierung
   - `episodic.last` — stärkster Beleg für abgeschlossene Konsolidierung
   - `consolidated`/`unconsolidated` — Counts, keine Timestamps
4. „Mnemosyne session end — running consolidation" = **Start-Marker**, kein Beweis für Abschluss.
5. Stored Timestamps exakt berichten; naiv (ohne Offset) → Host-Zeitzone separat nennen.

## 8. Profil-sichere manuelle Konsolidierung

Banken sind profil-scoped. `no_op` mit Zero-Counts kann falsches Profil/Store bedeuten, nicht „nichts zu tun".

1. Ziel-Profil auflösen (`~/.hermes` vs. `~/.hermes/profiles/<name>`), `memory.provider` bestätigen.
2. Before-Stats aufnehmen (working total, consolidated/unconsolidated, episodic total/last, vector count).
3. Skalen-Sanity: erwartete große Counts, aber Zeros/null Timestamps → Stopp, falsches Store.
4. **Ein** Konsolidierungslauf nur; kein paralleler Dry-Run (SQLite-Konkurrenz).
5. Langlaufende Läufe überwachen ohne Restart; Completion erst melden, wenn Prozess exit und After-Stats gelesen.
6. After-Stats vom **selben** Bank erfassen (exakte Zeitstempel, Status, Sessions/Items, Errors).

Direct-API-Fallback (nur wenn CLI/Profile-Wrapper das Ziel-Store nicht trifft):

```python
from pathlib import Path
from mnemosyne.core.memory import Mnemosyne
mem = Mnemosyne(
    session_id="manual_default_consolidation",
    db_path=Path.home() / ".hermes/mnemosyne/data/mnemosyne.db",
    bank="default",
)
before = mem.get_stats()
result = mem.sleep_all_sessions(dry_run=False, force=False)
after = mem.get_stats()
```

API-Signaturen können sich ändern — vor Ausführung installiertes Paket prüfen.

## 9. Diagnose-Kommandos

```bash
hermes memory status                                            # Provider + Plugin-Status
~/.hermes/venvs/mnemosyne/bin/mnemosyne stats                  # working vs. episodic
ls -lh ~/.hermes/mnemosyne/data/mnemosyne.db                   # DB-Größe
python3 -c "import os; print(sorted(k for k in os.environ if k.startswith('MNEMOSYNE')))"  # env ohne Secrets
# memory-Config anzeigen:
python3 - <<'PY'
import yaml, pathlib, json
cfg = yaml.safe_load((pathlib.Path.home()/'.hermes/config.yaml').read_text()) or {}
print(json.dumps(cfg.get('memory'), indent=2))
PY
```

## 10. USER.md / MEMORY.md auditen (auch bei aktivem Mnemosyne)

Hermes injiziert `~/.hermes/memories/USER.md` + `MEMORY.md` weiterhin bei Session-Start — als **kleine universelle Bootstrap-Schicht**, nicht als zweite Memory-DB:
- USER.md: stabile Identität, Kommunikationsstil, Präferenzen, Grenzen.
- MEMORY.md: stabile Topologie, Profile-Routing, dauerhafte Service-Pfade, Konventionen.
- Mnemosyne: sich entwickelnde, detaillierte oder gelegentlich relevante Kontexte.
- Skills/AGENTS.md: Prozeduren, Befehle, Troubleshooting.
- Session-History: abgeschlossene Arbeit, Versionen, Counts, Job-IDs.

Audit: Backup mit Timestamp → Duplikate entfernen (was Skills/Config/Mnemosyne bereits abdecken) → dynamische Fakten raus (live abfragen) → kompakt, `§`-Delimiter erhalten → beide Dateien nach dem Edit zurücklesen. Änderungen greifen erst in neuer Session (frozen Startup-Snapshot).
