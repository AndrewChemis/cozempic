# Design — Session-End Cost Accounting, Cache-Friendly Compression & Supply-Chain Hardening

## Overview

Three loosely-coupled changes. (1) and (2) are delivered through a single new
`SessionEnd` hook that invokes one new CLI command; (3)–(5) are isolated edits
to CI, packaging, and the auto-update gate. Nothing in existing hook behaviour
(`Stop`, `PreCompact`, `PostCompact`, `PostToolUse`, `SessionStart`) changes.

Why `SessionEnd` and not `Stop`: `Stop` fires after every assistant turn
(`src/cozempic/data/hooks.json:65`), so cost-logging or pruning there would run
dozens of times per session and could rewrite the transcript while it is still
live. `SessionEnd` fires once, at teardown — the correct point to total a
session and to slim a transcript that the next trigger will replay.

---

## Architecture

```
Claude Code session ends
        │  (stdin JSON: session_id, transcript_path, cwd, reason)
        ▼
SessionEnd hook  ──►  cozempic session-end
                         │
                         ├─ 1. read transcript ONCE (messages + costUSD/usage)
                         ├─ 2. cost accounting  ──► ~/.claude/cozempic-metrics/session-costs.jsonl
                         └─ 3. compression      ──► gentle prune, safe-write back to transcript
```

Ordering is deliberate: cost is computed from the in-memory messages **before**
the compression pass strips `costUSD`/`usage` (Requirement 2.8). A single load
serves both steps.

### New CLI command: `session-end`

- Reads the hook payload from stdin (mirrors `cmd_nudge`,
  `src/cozempic/cli.py:1235`).
- Resolves the transcript path from the payload; bails to exit 0 if missing /
  unreadable (Requirement 1.7).
- Loads messages once via `session.load_messages`.
- Calls `cost.record_session_cost(messages, payload)` then
  `compress.compress_on_end(path, messages, snapshot)`.
- Always exits 0; every failure path is swallowed and (optionally) logged via
  `COZEMPIC_DEBUG`, matching existing hook discipline.

The command is registered in `build_parser` (`src/cozempic/cli.py:1572`) and
dispatched like the other hook subcommands. It is intentionally a hook-only
command (no positional `session` arg) so it cannot be confused with `treat`.

---

## Components and interfaces

### Component A — Cost accounting (`src/cozempic/cost.py`, new)

```python
METRICS_DIR = Path.home() / ".claude" / "cozempic-metrics"
COST_FILE   = METRICS_DIR / "session-costs.jsonl"

def compute_cost(messages) -> CostResult:
    """Sum per-message costUSD; fall back to token usage when absent."""

def record_session_cost(messages, payload) -> None:
    """Compute + append one JSONL record. Idempotent per session_id. No-op on opt-out."""
```

- `costUSD` lives on the outer message object (it is in `gentle`'s
  `strip_outer` set, `src/cozempic/strategies/gentle.py:241`). Sum it across all
  messages that carry it.
- When absent, pull `extract_usage_tokens(messages)`
  (`src/cozempic/tokens.py:254`) and `detect_model(messages)` and set
  `cost_usd = null`, `cost_source = "unavailable"`.
- Idempotency (Requirement 1.6): before appending, scan the tail of
  `session-costs.jsonl` for an existing record with the same `session_id`; skip
  if found. (Tail scan is cheap; the file is one short line per session.)
- Opt-out (Requirement 1.8): return early if `COZEMPIC_COST_LOG_OFF` or
  `COZEMPIC_NO_TELEMETRY` is set. (This file is local-only, never transmitted —
  `NO_TELEMETRY` is honoured as a courtesy for users who want zero side files.)
- Atomic append: write via a tmp-file rewrite (read existing + append +
  `_atomic_write_text`) reusing `digest._atomic_write_text` semantics, OR an
  `O_APPEND` write under a short-lived lock. Given the file is tiny and writes
  are rare (once per session), read-modify-atomic-replace is simplest and
  crash-safe.

#### Record schema (`session-costs.jsonl`, one JSON object per line)

```json
{
  "session_id": "abc123…",
  "project": "/home/user/cozempic",
  "ended_at": "2026-06-17T12:34:56+00:00",
  "reason": "clear",
  "cost_usd": 0.4213,
  "cost_source": "costUSD",
  "tokens_total": 187432,
  "model": "claude-opus-4-8"
}
```

`cost_usd` is `null` and `cost_source` is `"unavailable"` on subscription
transcripts that carry no `costUSD`.

### Component B — Compression (`src/cozempic/compress.py`, new thin wrapper)

Reuses the existing prune pipeline rather than reimplementing it:

```python
def compress_on_end(path, messages, snapshot) -> CompressResult:
    if os.environ.get("COZEMPIC_SESSION_END_COMPRESS_OFF"):
        return CompressResult(skipped="opt-out")
    if below_floor(messages):                       # Req 2.6
        return CompressResult(skipped="below-floor")
    new_messages, _ = run_prescription(messages, PRESCRIPTIONS["gentle"], {})
    try:
        with _PruneLock(path):                      # Req 2.2/2.3
            save_messages(path, new_messages, create_backup=True, snapshot=snapshot)
    except PruneLockError:
        return CompressResult(skipped="locked")
    except PruneConflictError:
        return CompressResult(skipped="changed-mid-write")  # Req 2.4
```

- Prescription is fixed to `gentle` (Requirement 2.1) — `metadata-strip` +
  file-history dedup. No thinking-block removal, no content drop, so resume
  fidelity is preserved.
- `snapshot` is taken in `cmd_session_end` before load, exactly as `cmd_treat`
  does (`src/cozempic/cli.py:350`), to power append-conflict detection.
- The benefit floor (Requirement 2.6) avoids creating a backup for a 3-message
  session; suggested default ~50 KB transcript, overridable via
  `COZEMPIC_SESSION_END_COMPRESS_MIN_BYTES`.

### Component C — Hook wiring (`src/cozempic/data/hooks.json` + `plugin/hooks/hooks.json`)

Add a new `SessionEnd` block (both files are kept byte-identical by
`tests/test_hooks_sync.py`):

```jsonc
"SessionEnd": [{
  "matcher": "",
  "hooks": [{
    "type": "command",
    "command": "export COZEMPIC_NO_AUTO_INIT=1; printf %s \"$(cat)\" | { cozempic session-end 2>/dev/null || python3 -m cozempic session-end 2>/dev/null; } || true # cozempic-hook-schema=v14"
  }]
}]
```

- Bumps the embedded `cozempic-hook-schema` marker (currently `v13`) to `v14`
  so `init`/`doctor` re-sync installed hooks.
- Mirrors the defensive pattern of existing hooks: `COZEMPIC_NO_AUTO_INIT`,
  `cozempic … || python3 -m cozempic …` fallback, `|| true`, all output
  suppressed.

### Component D — GitHub Actions SHA pinning (`packaging/ci/publish.yml`)

Replace each floating tag with a pinned SHA + comment, e.g.:

```yaml
- uses: actions/checkout@<40-char-sha>            # v4.x.y
- uses: actions/setup-python@<40-char-sha>        # v5.x.y
- uses: actions/upload-artifact@<40-char-sha>     # v4.x.y
- uses: actions/download-artifact@<40-char-sha>   # v4.x.y
- uses: pypa/gh-action-pypi-publish@<40-char-sha> # release/v1 → vX.Y.Z
```

Implementation note: SHAs are resolved at implementation time from the upstream
repos' release tags (e.g. via the GitHub API / `pin-github-action`). Add a
short `packaging/ci/README` note or Dependabot config
(`package-ecosystem: github-actions`) to refresh pins deliberately
(Requirement 3.4). Confirm whether the file must move to
`.github/workflows/publish.yml` to be an active workflow (Requirement 3.3).

### Component E — Hashed build tooling (CI)

- Add `packaging/ci/build-requirements.txt` with exact, hashed pins for
  `build`, `setuptools`, `wheel`.
- Change the CI build step from `pip install --upgrade build` to
  `pip install --require-hashes -r packaging/ci/build-requirements.txt`.
- Align `pyproject.toml` `[build-system] requires` to the same exact versions.
- Document a refresh procedure (e.g. `pip-compile --generate-hashes`).

### Component F — Auto-update opt-in flip

Three call sites must agree (Requirement 5.4):

1. **`src/cozempic/updater.py` (`maybe_auto_update`, line 225)** — at the top,
   after the existing `COZEMPIC_NO_AUTO_UPDATE` / pin checks, add:
   `if not os.environ.get("COZEMPIC_AUTO_UPDATE"): <print throttled notice>; return`.
   The notice reuses `_get_latest_version` + `_upgrade_hint` and the existing
   24h `_should_check`/`_mark_checked` throttle so it stays quiet
   (Requirement 5.3). `COZEMPIC_NO_AUTO_UPDATE` and `COZEMPIC_PIN` continue to
   short-circuit earlier, so explicit-disable beats opt-in (Requirement 5.5).

2. **`src/cozempic/data/hooks.json:9` + `plugin/hooks/hooks.json` mirror** — the
   upgrade clause is currently gated `[ -z "$COZEMPIC_NO_AUTO_UPDATE" ] && [ -z
   "$COZEMPIC_PIN" ]`. Add the opt-in requirement: only run the
   `uv pip install --upgrade …` block when `COZEMPIC_AUTO_UPDATE` is non-empty
   AND the existing disables are unset.

3. **`npm/install.js` (`decideInstall`)** — `--upgrade` is currently added
   unless pinned/disabled. Change so `--upgrade` is added only when
   `COZEMPIC_AUTO_UPDATE` is set (and not pinned/disabled). First-time install
   (no existing version) still installs the package without `--upgrade`.

Optional (Requirement 5.7): when opt-in IS enabled, gate `_do_upgrade` behind a
publish-age check using the PyPI JSON `releases[version][].upload_time` already
fetched by `_get_latest_version`'s endpoint.

---

## Data models

| Artifact | Location | Lifetime |
| --- | --- | --- |
| `session-costs.jsonl` | `~/.claude/cozempic-metrics/` | persistent, append-only |
| compressed transcript | Claude Code project dir (in place) | overwrites, with `.bak` |
| nudge/metrics state | `~/.claude/cozempic-metrics/` (existing) | unchanged |

No new on-disk format beyond the JSONL record schema above.

---

## Error handling

Every new path runs inside a shutdown hook, so the invariant is **never raise,
always exit 0**:

- Missing/garbage stdin → return (mirrors `cmd_nudge`).
- Unreadable/missing transcript → return without writing.
- Metrics dir unwritable (read-only FS, quota) → swallow, optional
  `COZEMPIC_DEBUG` stderr line.
- Compression lock contended or mid-write conflict → skip the prune, still write
  cost; transcript left untouched.
- Network failure in the auto-update notice → silent (existing behaviour).

---

## Testing strategy

- **cost.py**: `costUSD` summation; subscription fallback (usage-only →
  `cost_usd=null`); idempotency on duplicate `session_id`; opt-out env;
  unwritable dir degrades silently; atomic-write crash-safety (reuse the
  `endswith`-patch pattern from `tests/test_atomic_writes_wave1.py`).
- **compress.py**: gentle prune shrinks a bloated fixture
  (`tests/fixtures/sessions/solo_bloated.jsonl`); lock contention → skip;
  append-conflict (snapshot mismatch) → no write; below-floor → skip; backup
  created on success.
- **ordering**: cost is recorded with non-null `costUSD` even though the
  subsequent compression strips `costUSD` (Requirement 2.8).
- **hooks sync**: extend `tests/test_hooks_sync.py` so the new `SessionEnd`
  block stays identical across `data/hooks.json` and `plugin/hooks/hooks.json`,
  and the schema marker bump is asserted.
- **updater opt-in**: default (no env) does not upgrade and prints at most one
  notice per 24h; `COZEMPIC_AUTO_UPDATE=1` restores upgrade; pin/disable still
  win. Extend the existing `decideInstall` unit test in npm for the new gate.
- **CI pinning**: a lint/test asserting every `uses:` in the workflow is a
  40-hex SHA (regex `@[0-9a-f]{40}`), to prevent regressions to floating tags.

---

## Decisions & open questions

- **Dollars vs tokens fallback** — DECIDED: record tokens + mark dollars
  unavailable rather than ship a price table now (Out of scope in requirements).
- **One command vs two** — DECIDED: a single `session-end` command does both, on
  one transcript load, so cost reads pre-compression fields.
- **OPEN (Req 3.3)** — is `packaging/ci/publish.yml` an active workflow or
  documentation? If active, it must move to `.github/workflows/`. Resolve before
  implementing Component D.
- **OPEN** — exact compression benefit floor default (50 KB proposed).
- **OPEN (Req 5.7)** — ship the publish-age window now or as a fast follow once
  the opt-in flip lands.
