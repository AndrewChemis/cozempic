# Design — Cache-Aware Compression & Supply-Chain Hardening

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
TRIGGER A (primary): guard daemon notices                TRIGGER B (fast path):
no new message for > cache TTL (default 300s)            explicit SessionEnd event
        │                                                        │ stdin JSON:
        │ (warm cache already expired → free to prune)           │ session_id, transcript_path,
        ▼                                                        ▼ cwd, reason
   compress_to_target(transcript)                        cozempic session-end
        │                                                        │
        ▼                                                        ▼
   prune gentle→standard→aggressive  ◄───────────────────────────┘
   only as far as needed to land ≤ target % of window;
   safe-write back (PruneLock + snapshot conflict guard)
```

(Cost logging is no longer part of this command — it moved to the separate
`claude/session-cost-logging` PR. Both triggers now do compression only.)

**Why idle-past-TTL is the right moment.** The hook is free (0 model tokens),
but a compaction event is not — it generates a ~12% summary at output rates plus
a prefix read + re-cache. Pruning when the cache has *already* expired forfeits
no warm hit, and the slimmer transcript keeps a later re-trigger under the
compaction threshold. See `cost-analysis.md`.

### New CLI command: `session-end`

- Reads the hook payload from stdin (mirrors `cmd_nudge`,
  `src/cozempic/cli.py:1235`).
- Resolves the transcript path from the payload; bails to exit 0 if missing /
  unreadable.
- Loads messages once via `session.load_messages`.
- Calls `compress.compress_to_target(path, messages, snapshot)`.
- Always exits 0; every failure path is swallowed and (optionally) logged via
  `COZEMPIC_DEBUG`, matching existing hook discipline.

The command is registered in `build_parser` (`src/cozempic/cli.py:1572`) and
dispatched like the other hook subcommands. It is intentionally a hook-only
command (no positional `session` arg) so it cannot be confused with `treat`.

---

## Components and interfaces

### Component A — Cost accounting → MOVED

Split out to `claude/session-cost-logging` (`.kiro/specs/session-cost-logging/spec.md`).
Not part of this design.

### Component B — Compression: prune-to-target (`src/cozempic/compress.py`, new)

Reuses the existing prune pipeline, but escalates prescriptions until the
transcript lands under a target headroom rather than applying a fixed strength:

```python
TARGET_PCT = 0.55   # land at/below the guard's hard1 tier (configurable)
LADDER = ("gentle", "standard", "aggressive")

def compress_to_target(path, messages, snapshot) -> CompressResult:
    if os.environ.get("COZEMPIC_SESSION_END_COMPRESS_OFF"):
        return CompressResult(skipped="opt-out")
    if below_floor(messages) or at_or_below_target(messages):   # Req 2.9
        return CompressResult(skipped="already-small")
    window = detect_context_window(messages)
    target_tokens = int(window * target_pct())
    new_messages = messages
    for rx in LADDER:                                # Req 2.1 — escalate as needed
        new_messages, _ = run_prescription(messages, PRESCRIPTIONS[rx], {})
        if estimate_session_tokens(new_messages).total <= target_tokens:
            break                                    # gentle is the floor; stop early
    try:
        with _PruneLock(path):                       # Req 2.6/2.7
            save_messages(path, new_messages, create_backup=True, snapshot=snapshot)
    except PruneLockError:
        return CompressResult(skipped="locked")
    except PruneConflictError:
        return CompressResult(skipped="changed-mid-write")
```

- **Prune-to-target, not fixed-gentle** (Requirement 2.1): a 5% gentle trim on a
  transcript already at ~90% of the window still gets compacted on the next
  re-trigger — money/latency spent for nothing. Escalating only as far as needed
  keeps the lightest touch that actually clears the compaction threshold.
- `gentle` is the floor (always at least metadata-strip + file-history dedup);
  `aggressive` is reached only when the session is genuinely huge.
- Target defaults to 55% of the detected window (`detect_context_window`,
  `src/cozempic/tokens.py:176`), overridable via
  `COZEMPIC_SESSION_END_COMPRESS_TARGET_PCT`. This mirrors the guard's hard1
  tier so the end-of-session prune lands where mid-session reloads already aim.
- `snapshot` powers append-conflict detection exactly as `cmd_treat` does
  (`src/cozempic/cli.py:350`).
- Floor skip (`COZEMPIC_SESSION_END_COMPRESS_MIN_BYTES`, ~50 KB) avoids a backup
  for a trivial session.

### Component B2 — Triggers: idle-past-cache-TTL is primary

Two trigger paths drive Component B; the idle one is primary because the doc
research found `SessionEnd` is unreliable on `/exit` and `/clear`:

1. **Idle-past-cache-TTL (primary, Req 2.2/2.2a/2.2b/2.4/2.5).** The guard daemon
   (`cozempic guard --daemon`, already spawned at SessionStart and already
   polling the transcript — `src/cozempic/guard.py`) gains an idle check that
   tracks last-activity (transcript mtime / last main-chain message).

   - **Auto-derive the threshold from the cache-TTL mode** (Req 2.2a) by the
     documented precedence: `COZEMPIC_CACHE_TTL_SECONDS` > `FORCE_PROMPT_CACHING_5M`
     (→5-min) > `DISABLE_PROMPT_CACHING*` (off) > subscription auth (→1-hour,
     automatic/free) > API-key + `ENABLE_PROMPT_CACHING_1H` (→1-hour) > API-key
     default (→5-min). The modes differ in *write* price (1.25× at 5-min, 2.0× at
     1-hour; reads 0.1× both), so the wait must match the mode.
   - **Safe default = the LONGER window** (Req 2.2c): when the mode is uncertain,
     assume 1-hour. Firing too early (assume 5-min, real 1-hour) prunes a still-warm
     cache and forfeits 0.1× hits; firing too late only delays the benefit. The
     known silent downgrades (plan-overage→credits, subagents, the reported
     #46829 regression) aren't visible in env, so the robust enhancement is to
     **measure the effective TTL** from transcript `usage` (cache_read vs
     cache_creation across turn gaps) rather than trust env alone (Req 2.2d).
   - **Telemetry/privacy vars are NOT caching signals** (Req 2.2e):
     `DISABLE_TELEMETRY`, `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`,
     `DO_NOT_TRACK`, `COZEMPIC_NO_TELEMETRY` — docs confirm they don't affect the
     cache, so the detector must ignore them.
   - **Fire just after expiry, never before** (Req 2.2b): WHEN
     `now - last_activity > threshold + ε` AND transcript above target AND no
     work in flight (`detect_in_flight`) THEN run `compress_to_target`. Pruning
     rewrites the prefix and would invalidate a still-warm cache; firing before
     expiry could forfeit a 0.1× hit for no gain. After expiry there is **no warm
     hit left to forfeit**, so pruning is downside-free and pre-positions the
     next resume to re-ingest a slim file → a cheaper (but still unavoidable)
     rebuild, and native LLM compaction pre-empted.

   Note the scope of the win: the server-side cache cannot be saved/restored by a
   local tool, so the post-eviction rebuild is unavoidable — compression makes it
   *cheaper* (smaller prefix) and *lossless* (no LLM summary), it does not skip it.

2. **Explicit SessionEnd (fast path, Req 2.3).** The `SessionEnd` hook runs
   `cozempic session-end`, which calls `compress_to_target` immediately — no idle
   wait. Covers clean exits where the event does fire.

Both paths converge on the same `compress_to_target` + safe-write, so behaviour
is identical regardless of which fires. The idle path means coverage never
depends on `SessionEnd` firing (Req 2.5).

### Component B3 — Resume backstop: do the compaction ourselves (Req 2.11–2.15)

Wires into the **existing** `SessionStart` hook (`src/cozempic/data/hooks.json:9`
+ mirror), which already runs on resume and already contains the
`guard --reload-self` machinery (used today on the auto-update version-change
path). On `SessionStart` with matcher `resume` for a transcript above target:

```
SessionStart(resume), same session_id
        │
        ├─ transcript ≤ target?  ──► no-op (already pruned at idle/SessionEnd)   # Req 2.13
        │
        └─ above target ──► compress_to_target(transcript)                        # Req 2.11
                                   │
                                   ├─ hook runs BEFORE ingestion ──► Claude loads slim file. Done.
                                   └─ hook runs AFTER ingestion  ──► guard --reload-self           # Req 2.12
                                          re-enter against pruned file (small mechanical
                                          cache-creation) instead of native compaction
                                          (LLM summary over the large prefix)
```

**Do not double-pay the resume rebuild.** If the hook runs AFTER ingestion,
Claude has already reprocessed the un-pruned prefix on this resume — that rebuild
is sunk. `reload-self` would then add a SECOND (smaller) rebuild, so in the
post-ingestion case the backstop prunes the file for the NEXT resume and skips
`reload-self` for cost purposes (it becomes an optional tradeoff that only pays
off via cheaper subsequent cache-reads on long post-resume sessions). The current
resume's rebuild is reduced ONLY when the hook runs pre-ingestion. This is why
pruning DURING idle (daemon alive, before resume) is the high-value path and does
not depend on the hook-timing question — the resume backstop is a secondary net.

**Sequencing — the one thing to confirm.** Whether the `SessionStart(resume)`
hook fires before or after Claude ingests the transcript decides which branch is
the common case. The design works either way (in-place when before;
`reload-self` when after), but the answer tunes the default. This is the single
open question gating Component B3 — resolve from docs before implementing. The
existing reload-self auto-update path strongly implies "after" (you can't mutate
an already-loaded context without re-entry), which is why `reload-self` is the
safe default.

**Relationship to idle/SessionEnd prune.** End-of-interaction pruning is primary;
the resume backstop is the guarantee. If end-time pruning ran, the resume check
no-ops (Req 2.13). If it didn't (the `/exit`, `/clear`, or container-killed
gaps), the backstop ensures the resumed session still never hits native
compaction.

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
| compressed transcript | Claude Code project dir (in place) | overwrites, with `.bak` |
| nudge/metrics state | `~/.claude/cozempic-metrics/` (existing) | unchanged |

No new on-disk format. (`session-costs.jsonl` belongs to the separate
session-cost-logging feature.)

---

## Error handling

Every new path runs inside a shutdown hook, so the invariant is **never raise,
always exit 0**:

- Missing/garbage stdin → return (mirrors `cmd_nudge`).
- Unreadable/missing transcript → return without writing.
- Metrics dir unwritable (read-only FS, quota) → swallow, optional
  `COZEMPIC_DEBUG` stderr line.
- Compression lock contended or mid-write conflict → skip the prune; transcript
  left untouched.
- Network failure in the auto-update notice → silent (existing behaviour).

---

## Testing strategy

- **compress.py**: gentle prune shrinks a bloated fixture
  (`tests/fixtures/sessions/solo_bloated.jsonl`); lock contention → skip;
  append-conflict (snapshot mismatch) → no write; below-floor → skip; backup
  created on success.
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

- **Prune strength** — DECIDED: prune-to-target (escalate gentle→standard→
  aggressive to land ≤ target % of window), not a fixed `gentle` trim. A fixed
  gentle pass is wasted on a transcript already near the compaction threshold.
  Target defaults to 55% (guard hard1 tier).
- **Compression trigger** — DECIDED: idle-past-cache-TTL via the guard daemon is
  the *primary* trigger; explicit `SessionEnd` is a fast path. The doc research
  showed `SessionEnd` is unreliable on `/exit`/`/clear`, so coverage must not
  depend on it. Pruning is timed to cache expiry so no warm hit is forfeited.
- **Dollars vs tokens fallback** — DECIDED: record tokens + mark dollars
  unavailable rather than ship a price table now (Out of scope in requirements).
- **Cost logging split out** — DECIDED: session-end cost-to-file is now a separate
  feature (`claude/session-cost-logging`); this `session-end` command does
  compression only.
- **OPEN (Req 3.3)** — is `packaging/ci/publish.yml` an active workflow or
  documentation? If active, it must move to `.github/workflows/`. Resolve before
  implementing Component D.
- **Cache-TTL idle window** — DECIDED (verified against official docs): auto-derive
  by the precedence in Req 2.2a, fire just *after* expiry, default to the LONGER
  window when uncertain (safe error direction), ignore telemetry/privacy vars.
  Confirmed: subscription auth = automatic free 1-hour TTL;
  `FORCE_PROMPT_CACHING_5M` downgrades any auth; plan-overage/subagents silently
  use 5-min. OPEN sub-items: (a) exact subscription-vs-API-key detection from the
  hook env; (b) whether to ship the empirical TTL-measurement enhancement (Req
  2.2d) now or rely on the safe 1-hour default first.
- **OPEN (Req 5.7)** — ship the publish-age window now or as a fast follow once
  the opt-in flip lands.
