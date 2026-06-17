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

2.2. WHEN no new activity has been written to the transcript for longer than the
detected prompt-cache TTL AND the transcript is above target THEN compression
SHALL fire. This is the **primary, exit-path-independent** trigger.

2.2a. The idle threshold SHALL be **auto-derived from the cache-TTL mode** using
this precedence (highest first), per the official prompt-caching docs:
  1. `COZEMPIC_CACHE_TTL_SECONDS` (explicit user override) — wins over all below.
  2. `FORCE_PROMPT_CACHING_5M=1` → 5-min mode (~300s), any auth incl. subscription.
  3. `DISABLE_PROMPT_CACHING[_OPUS|_SONNET|_HAIKU|_FABLE]=1` → caching off; the
     cache-forfeit concern is moot, but the idle prune MAY still run to pre-empt
     native compaction and slim the reload.
  4. Claude subscription auth → 1-hour mode (~3600s, automatic, free).
  5. API key / Bedrock / Vertex / Foundry with `ENABLE_PROMPT_CACHING_1H=1` →
     1-hour (~3600s). (`ENABLE_PROMPT_CACHING_1H` has no effect on subscription.)
  6. API key / third-party default → 5-min mode (~300s).

2.2c. WHERE the TTL mode cannot be positively determined THEN the threshold SHALL
default to the **longer (1-hour)** window. Firing too late merely delays
pre-positioning (safe); firing too early — assuming 5-min when the real TTL is
1-hour — would prune while the cache is still warm and forfeit 0.1× hits.
Over-estimating the TTL is the safe error direction.

2.2d. The detector SHALL account for documented silent downgrades that env vars
do NOT signal: (a) a subscription over its plan limit drawing on usage credits
drops to 5-min; (b) subagent contexts use 5-min even on a subscription; (c)
reported server-side TTL changes (e.g. issue #46829, unverified). Because env
inference is therefore best-effort, an enhancement (Req 2.2c default keeps it
safe meanwhile) is to **measure the effective TTL empirically** from transcript
`usage` (cache_read vs cache_creation across turn gaps) and prefer the measured
value.

2.2e. Telemetry / privacy env vars (`DISABLE_TELEMETRY`,
`CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`, `DO_NOT_TRACK`, and Cozempic's own
`COZEMPIC_NO_TELEMETRY`) SHALL NOT be used as caching signals — per docs they do
not affect prompt caching or the TTL.

2.2b. Compression SHALL fire **just after** the TTL elapses (threshold = TTL + a
small margin), never before it. Rationale: pruning rewrites the prefix and
invalidates the cache; firing before expiry would forfeit a still-warm 0.1× hit
if the user resumes in the final second, for no benefit. Once the cache has
expired there is nothing left to forfeit, and an idle session is in no hurry.

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

2.12. The backstop's effect on the CURRENT resume depends on hook-vs-ingestion
ordering, and it SHALL NOT cause a double-paid rebuild:
  - IF the hook runs BEFORE Claude ingests the transcript THEN pruning in place
    reduces this resume's rebuild directly (Claude ingests the slim file).
  - IF the hook runs AFTER ingestion THEN this resume's full rebuild is already
    sunk; the backstop SHALL prune the file to benefit the NEXT resume (and
    cheaper subsequent cache-reads) and SHALL NOT trigger `guard --reload-self`
    purely to cut the already-paid resume cost — that would add a second
    (smaller) rebuild. `reload-self` is then an OPTIONAL tradeoff that pays off
    only via cheaper subsequent reads on long post-resume sessions.

2.12a. Because of 2.12, the **high-value path is pruning DURING idle** (daemon
alive, before resume), which is independent of hook ordering. The resume backstop
is a secondary safety net for when idle/SessionEnd pruning did not run (daemon
not alive, e.g. the terminal was fully closed).

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
