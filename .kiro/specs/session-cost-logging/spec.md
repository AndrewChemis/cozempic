# Spec — Write total session cost to a local file on session end

Standalone feature, split out of the `session-cost-caching` investigation (PR for
that work is separate). This is independent session-cost **telemetry to a file** —
unrelated to cache-rebuild reduction or the status-line nudge.

## Overview

On session end, record the total cost of the session to a durable file **outside
the repository** so spend can be tracked over time without polluting the project
tree. Uses a new `SessionEnd` hook (distinct from the per-turn `Stop` hook) and a
new `cozempic` subcommand. Fire-and-forget: a `SessionEnd` hook's stdout is not fed
to the model, so this costs **0 model tokens**.

## Grounding in the codebase

- Claude Code writes a per-message **`costUSD`** field into the transcript JSONL —
  `strategies/gentle.py` strips it as metadata (`strip_outer = {"costUSD", …}`).
- Exact token usage is available via `tokens.extract_usage_tokens`.
- Established outside-the-repo metrics dir: `~/.claude/cozempic-metrics/` (already
  used for `nudge-state.json`).
- stdin parsing pattern: `cmd_nudge` (`cli.py`).
- Atomic write helper: `digest._atomic_write_text`.

## Requirements (EARS)

1. WHEN a session ends THEN Cozempic SHALL be invoked via a `SessionEnd` hook
   receiving the hook payload (`session_id`, `transcript_path`, `cwd`, `reason`) on
   stdin.
2. WHEN the transcript has per-message `costUSD` THEN the total SHALL be the sum of
   those fields.
3. IF no `costUSD` is present (subscription plans) THEN the system SHALL record exact
   token usage (`extract_usage_tokens`) AND mark the dollar total **unavailable**
   rather than reporting `$0.00`.
4. WHEN a total is computed THEN the system SHALL append one JSON record to
   `~/.claude/cozempic-metrics/session-costs.jsonl` with at least: `session_id`,
   `project`, `ended_at` (ISO-8601 UTC), `reason`, `cost_usd` (nullable),
   `cost_source` (`"costUSD"` | `"unavailable"`), `tokens_total`, `model`.
5. WHERE the metrics dir is absent THEN it SHALL be created (parents included).
6. WHEN the same `session_id` ends more than once THEN a duplicate record SHALL NOT
   be written.
7. IF the transcript path is missing/empty/unreadable THEN the command SHALL exit 0
   without writing and without raising (it runs in a shutdown hook).
8. WHERE `COZEMPIC_COST_LOG_OFF` or `COZEMPIC_NO_TELEMETRY` is set THEN nothing SHALL
   be written.
9. WHEN appending THEN the write SHALL be atomic/crash-safe (reuse
   `digest._atomic_write_text` semantics).

### Record schema (`session-costs.jsonl`, one object per line)

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

## Design

- **`src/cozempic/cost.py`** (new):
  - `compute_cost(messages) -> CostResult` — sum per-message `costUSD`; fallback to
    `extract_usage_tokens` + `detect_model`, with `cost_usd=None`,
    `cost_source="unavailable"`.
  - `record_session_cost(messages, payload)` — build record, honor opt-outs, create
    `~/.claude/cozempic-metrics/`, append atomically; per-`session_id` idempotency via
    a tail-scan of the file (one short line per session).
- **`cmd_session_end(args)`** in `cli.py` + subparser — read stdin payload,
  `load_messages` once, call `record_session_cost`, always exit 0; swallow every
  error (optional `COZEMPIC_DEBUG` stderr).
- **Hook wiring** — add a `SessionEnd` block to `src/cozempic/data/hooks.json` and the
  `plugin/hooks/hooks.json` mirror (kept identical by `tests/test_hooks_sync.py`); bump
  the `cozempic-hook-schema` marker so `init`/`doctor` re-sync.

## Tasks

- [ ] `cost.py`: `compute_cost` (costUSD sum + token fallback) + `record_session_cost`
  (record, opt-outs, dir create, atomic append) + per-`session_id` idempotency.
- [ ] `cmd_session_end` + subparser registration; always exit 0.
- [ ] `SessionEnd` hook block in both `hooks.json` files; schema-marker bump;
  extend `tests/test_hooks_sync.py`.
- [ ] Tests: costUSD sum; usage-only fallback (`cost_usd=null`); duplicate-session
  no-op; opt-out env; unwritable-dir degrade; atomic-write crash-safety; integration
  (payload → record written).
- [ ] Docs/README: the metrics file location + opt-outs.

## Notes / out of scope

- `SessionEnd` reportedly fires unreliably on `/exit` (anthropics/claude-code#17885)
  and `/clear` (#6428) — acceptable for best-effort cost logging; not the only net.
- A model→USD **price table** is out of scope (only needed to report dollars when
  `costUSD` is absent). This records tokens + marks dollars unavailable instead;
  a price table could be a follow-up.
- Distinct from the **status-line cost display** (live `cost.total_cost_usd` shown in
  the status line) tracked under the `session-cost-caching` work — that's a visible
  widget, not a file log.
