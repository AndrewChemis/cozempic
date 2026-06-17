# Requirements — Session-End Cost Accounting, Cache-Friendly Compression & Supply-Chain Hardening

## Introduction

This spec covers three capabilities investigated together on branch
`claude/session-cost-caching`:

1. **Session-end cost accounting** — on true session end, write the total cost
   of the session to a durable file outside the repository.
2. **Cache-aware session compression** — when an interactive session goes idle
   past the prompt-cache TTL (or ends explicitly), run a Cozempic prune that
   takes the transcript to a safe margin below the auto-compaction threshold, so
   that a later re-trigger replays a slim prefix AND avoids a costly compaction
   event. The trigger is timed to cache expiry: once the warm cache is forfeit,
   pruning costs nothing and pre-positions a cheaper re-trigger.
3. **Supply-chain hardening** — pin GitHub Actions and Python build tooling to
   immutable SHAs, and change the auto-update default from opt-out to opt-in so a
   compromised PyPI release cannot silently propagate to every user.

### Grounding in the current codebase

- The only "end-ish" hook today is `Stop` (`src/cozempic/data/hooks.json:65`),
  which fires after **every** assistant turn, not at true session end. Claude
  Code exposes a distinct `SessionEnd` event that Cozempic does not yet wire.
- Claude Code writes a per-message `costUSD` field into the transcript JSONL;
  the `metadata-strip` strategy strips it
  (`src/cozempic/strategies/gentle.py:241`). Exact token usage is available via
  `extract_usage_tokens` (`src/cozempic/tokens.py:254`).
- An established "outside the repo" metrics location already exists:
  `~/.claude/cozempic-metrics/` (used at `src/cozempic/cli.py:1285` for
  `nudge-state.json`).
- Pruning machinery (`cmd_treat`, the `gentle` prescription, `save_messages`
  with `_PruneLock` + snapshot append-conflict detection) already exists
  (`src/cozempic/cli.py:345`).
- Auto-update runs `uv pip install --upgrade cozempic` on **every** SessionStart
  unless opted out (`src/cozempic/data/hooks.json:9`;
  `src/cozempic/updater.py:225`). Opt-outs today: `COZEMPIC_NO_AUTO_UPDATE`,
  `COZEMPIC_PIN`.
- CI uses floating action tags (`packaging/ci/publish.yml:34,35,49,62,67`) and
  floating build tooling (`pyproject.toml:2`, `pip install --upgrade build`).

---

## Requirement 1 — Session-end cost accounting

**User story:** As a Cozempic user, I want the total cost of each Claude Code
session recorded to a durable file outside my repository when the session ends,
so that I can track spend over time without polluting my project tree.

### Acceptance criteria

1.1. WHEN a Claude Code session ends THEN Cozempic SHALL be invoked via a
`SessionEnd` hook receiving the hook payload (`session_id`, `transcript_path`,
`cwd`, `reason`) on stdin.

1.2. WHEN the cost command runs AND the transcript contains per-message
`costUSD` fields THEN the system SHALL compute the session total as the sum of
those fields.

1.3. IF the transcript contains no `costUSD` fields (e.g. subscription plans)
THEN the system SHALL record the exact token usage (input, output,
cache-creation, cache-read) from `extract_usage_tokens` AND mark the dollar
total as unavailable rather than reporting `$0.00`.

1.4. WHEN a session total is computed THEN the system SHALL append one JSON
record to `~/.claude/cozempic-metrics/session-costs.jsonl` containing at least:
`session_id`, `project`, `ended_at` (ISO-8601 UTC), `cost_usd` (nullable),
`cost_source` (`"costUSD"` | `"unavailable"`), `tokens_total`, and `model`.

1.5. WHERE the metrics directory does not exist THEN the system SHALL create it
(parents included) before writing.

1.6. WHEN the same `session_id` ends more than once (duplicate `SessionEnd`
delivery) THEN the system SHALL NOT write a duplicate record for that session.

1.7. IF the transcript path is missing, empty, or unreadable THEN the system
SHALL exit 0 without writing and without raising, so the hook never disrupts
shutdown.

1.8. WHERE `COZEMPIC_NO_TELEMETRY` or a dedicated cost opt-out
(`COZEMPIC_COST_LOG_OFF`) is set THEN the system SHALL NOT write the cost file.

1.9. WHEN the append occurs THEN the write SHALL be atomic/crash-safe (tmp +
fsync + replace, or append-with-lock) consistent with existing Cozempic write
discipline (`digest._atomic_write_text`).

---

## Requirement 2 — Cache-aware compression: triggers and prune-to-target

**User story:** As a Claude Code user, I want my transcript compressed when an
interactive session goes idle past the prompt-cache TTL (or ends), so that if I
later re-trigger, the replayed prefix is small and a costly compaction event is
avoided. Because the warm cache is already gone at that point, pruning costs me
nothing in forfeited cache hits.

### Rationale — why this is economically sound

A `SessionEnd`/idle hook adds **0 model tokens** (its stdout is not fed to the
model — see `design.md` and `cost-analysis.md`). A compaction event is **not**
free: it generates a summary ≈12% of the prefix at output rates, plus a prefix
read and a re-cache, plus downstream rework from fidelity loss. So the unit of
savings is an **avoided compaction event**, not marginal cache-read shavings —
and since the trigger is free, there is no break-even to clear.

### Acceptance criteria

2.1. WHEN compression runs THEN it SHALL prune the transcript to a configurable
**target headroom below the auto-compaction threshold** (e.g. land at ≤55% of
the context window — the guard's hard1 tier), escalating prescriptions
`gentle → standard → aggressive` only as far as needed to reach the target, with
`gentle` (metadata-strip + file-history dedup) as the **floor**. A fixed
`gentle`-only trim is insufficient when the transcript already sits near the
compaction threshold.

2.2. WHEN no new user message has arrived within the prompt-cache TTL
(configurable via `COZEMPIC_CACHE_TTL_SECONDS`, default 300s) after the last
assistant turn AND the transcript is above target THEN compression SHALL fire.
This is the **primary, exit-path-independent** trigger.

2.3. WHEN an explicit `SessionEnd` event fires THEN compression SHALL also run
immediately as a clean-exit fast path, without waiting out the idle timer.

2.4. WHILE a session is actively in use (a user message arrived within the cache
TTL) THEN compression SHALL NOT fire, so an in-flight warm cache is never
invalidated mid-conversation.

2.5. WHERE `SessionEnd` does not fire reliably for an exit path (documented gaps
on `/exit` and `/clear`) THEN the idle-timeout trigger (2.2) SHALL still
compress the session, so coverage does not depend on `SessionEnd` firing.

2.6. WHEN compression writes THEN it SHALL reuse the existing safe-write path
(`save_messages` with `_PruneLock` and snapshot-based append-conflict
detection) so a concurrent guard prune cannot corrupt the file.

2.7. IF another prune cycle holds the per-session lock, OR the transcript
changed between snapshot and write (Claude appended new lines) THEN compression
SHALL abort cleanly (no write, exit 0).

2.8. WHEN compression succeeds THEN the system SHALL create a backup of the
pre-compression transcript consistent with `save_messages(create_backup=True)`.

2.9. WHERE the transcript is already at/below target, smaller than a configurable
floor (`COZEMPIC_SESSION_END_COMPRESS_MIN_BYTES`), or the opt-out
(`COZEMPIC_SESSION_END_COMPRESS_OFF`) is set THEN compression SHALL be skipped.

2.10. WHEN compression strips exact-usage metadata THEN cost accounting
(Requirement 1) SHALL read `costUSD`/usage **before** the compression pass runs,
so the two steps do not race over the same fields.

### Resume backstop — pre-empt native compaction

The principle: Claude's native auto-compaction is an **LLM summarization call**
(output-rate tokens, lossy). Cozempic's prune is a **mechanical file operation**
(zero LLM, structure-preserving). So instead of letting native compaction fire
on a resumed-but-bloated session, Cozempic does the compaction itself, cheaper,
before Claude's does.

2.11. WHEN a session is resumed (Claude Code `SessionStart` hook, matcher
`resume`) AND its persisted transcript is above the compaction target THEN
Cozempic SHALL prune it to target as a backstop, so the resumed session loads
slim and native auto-compaction does not fire.

2.12. IF the resumed process has already ingested the un-pruned transcript by the
time the `SessionStart` hook runs (so an in-place edit cannot affect the current
context — the same constraint that makes Cozempic use `reload` today) THEN the
backstop SHALL fall back to `guard --reload-self` to re-enter against the pruned
transcript. The re-ingest is a cache-creation over the **smaller** pruned prefix
(mechanical, no LLM), which SHALL be cheaper than native compaction summarizing
the **larger** prefix.

2.13. WHERE the resumed transcript is already at/below target (e.g. it was
pruned at idle/SessionEnd) THEN the resume backstop SHALL no-op.

2.14. The resume backstop SHALL key on `session_id` so it acts only on a genuine
resume of the same session, never on a fresh `startup`.

2.15. The resume backstop SHALL honour the same opt-out
(`COZEMPIC_SESSION_END_COMPRESS_OFF`) and safe-write guards
(`_PruneLock`, snapshot conflict) as the idle/SessionEnd path.

---

## Requirement 3 — Pin GitHub Actions to immutable SHAs

**User story:** As a maintainer, I want every GitHub Action pinned to a full
commit SHA, so that a compromised or retagged action cannot alter our
publish pipeline.

### Acceptance criteria

3.1. WHEN the publish workflow references a third-party action THEN each `uses:`
SHALL be pinned to a full 40-character commit SHA with a trailing `# vX.Y.Z`
comment for human readability.

3.2. The set to pin SHALL include at minimum `actions/checkout`,
`actions/setup-python`, `actions/upload-artifact`, `actions/download-artifact`,
and `pypa/gh-action-pypi-publish` (`packaging/ci/publish.yml:34,35,49,62,67`).

3.3. WHERE the workflow is intended to execute, it SHALL live at
`.github/workflows/publish.yml`; the spec SHALL resolve whether
`packaging/ci/publish.yml` is the active location or documentation-only.

3.4. WHEN actions are pinned THEN a documented, repeatable process (or
Dependabot/`pin-github-action` config) SHALL exist to refresh the SHAs on a
deliberate cadence rather than floating automatically.

---

## Requirement 4 — Pin Python build tooling to hashes

**User story:** As a maintainer, I want build-time Python dependencies pinned
with hashes, so that the artifact published to PyPI is built from a verified
toolchain.

### Acceptance criteria

4.1. WHEN the package is built in CI THEN build dependencies (`build`,
`setuptools`, `wheel`) SHALL be installed from a pinned, hashed requirements
file using `pip install --require-hashes`.

4.2. WHERE `pyproject.toml` declares `[build-system] requires`
(`pyproject.toml:2`) THEN those entries SHALL specify exact versions consistent
with the hashed CI lockfile.

4.3. The spec SHALL note that cozempic has **no runtime dependencies**, so
hashing is scoped to build/CI tooling only.

4.4. WHEN a hashed lockfile is added THEN a documented refresh procedure SHALL
exist so pins are updated deliberately, not silently.

---

## Requirement 5 — Make auto-update opt-in (flip the default)

**User story:** As a security-conscious user, I want Cozempic to NOT upgrade
itself automatically by default, so that a compromised PyPI release cannot reach
my machine within one session without my action.

### Acceptance criteria

5.1. WHEN a session starts AND no explicit opt-in is set THEN Cozempic SHALL NOT
run `pip/uv pip install --upgrade cozempic` and SHALL NOT auto-upgrade in
`updater.maybe_auto_update`.

5.2. WHERE `COZEMPIC_AUTO_UPDATE=1` is set THEN automatic upgrade behaviour
SHALL be enabled (restoring today's default for users who want it).

5.3. WHEN auto-update is disabled by default AND a newer version exists THEN the
system SHALL, at most once per 24h, print a non-blocking notice with the manual
upgrade command for the detected install method (`updater._upgrade_hint`).

5.4. The opt-in gate SHALL be honoured identically across ALL ingress paths:
the SessionStart shell hook (`src/cozempic/data/hooks.json:9` and the
`plugin/hooks/hooks.json` mirror), the Python updater
(`src/cozempic/updater.py:225`), and `npm/install.js` (`decideInstall`).

5.5. WHEN `COZEMPIC_NO_AUTO_UPDATE` or `COZEMPIC_PIN` is set THEN auto-update
SHALL remain disabled regardless of `COZEMPIC_AUTO_UPDATE` (explicit
disable/pin always wins over opt-in).

5.6. WHEN the default flips THEN README and any onboarding copy SHALL document
the new default and the `COZEMPIC_AUTO_UPDATE=1` opt-in.

5.7. WHERE auto-update IS enabled via opt-in THEN the system SHOULD additionally
support a publish-age window (refuse releases younger than N hours) so a
malicious publish that is later yanked has a bounded blast radius.

---

## Out of scope

- A model-by-model USD price table (would be required only if cost must be
  reported in dollars when `costUSD` is absent). Tracked as a follow-up; this
  spec records token totals and marks dollars unavailable instead.
- Signing/attestation verification of downloaded wheels beyond the publish-age
  window (future hardening).
- Changing the `Stop`, `PreCompact`, `PostCompact` hook behaviour; this spec
  adds a new `SessionEnd` hook and does not alter existing ones.
