# Feature Spec — `cozempic statusline` (nudge + cost in the Claude Code status line)

The one clearly-feasible, high-value feature to come out of this investigation.
It replaces the unreliable Stop-hook `systemMessage` nudge (§0 of `future-work.md`)
with the Claude Code **status line** — a first-class, reliably-rendered surface —
and folds in the original cost-display ask (Requirement 1) at the same time.

## Why the status line

- **Reliable where `systemMessage` is broken** (#50542/#40380). Shown at the `❯`
  prompt — exactly when a user decides to reload, or returns from idle.
- **Zero token cost, never model-visible** — a passive FYI, no risk of the model
  acting on it.
- **The payload does the work**: the status-line stdin JSON already includes
  `context_window.used_percentage`, `current_usage` (cache_read/cache_creation
  split), `cost.total_cost_usd`, `transcript_path`, `model`, `exceeds_200k_tokens`,
  `rate_limits`. — code.claude.com/docs/en/statusline
- Known limitation #50679 (hidden during long *task execution*) is irrelevant —
  we surface at the prompt, not mid-task.

## Requirements (EARS)

R1. The system SHALL provide a `cozempic statusline` command that reads the
Claude Code status-line JSON on stdin and prints a single-line status segment to
stdout.

R2. WHEN `context_window.used_percentage` is present THEN the command SHALL render
context usage (e.g. `↯ 62%`); WHERE it is null/absent (pre-first-API-call) THEN the
command SHALL fall back to `quick_token_estimate(transcript_path)` and render the
derived percentage, or omit the segment if neither is available.

R3. WHEN `cost.total_cost_usd` is present THEN the command SHALL render running
session cost (e.g. `$0.34`), satisfying the original cost-display ask in a visible
place. (No price table needed — Claude Code supplies the figure.)

R4. WHEN usage crosses a nudge tier (reusing the existing 25/55/80 tiers and any
`COZEMPIC_NUDGE_PCTS` / per-session tier override) THEN the command SHALL append a
colorized reload nudge (e.g. `⚠ /cozempic reload`), yellow ≥ the mid tier, red ≥
the high tier. ANSI color and emoji are supported by the status line.

R5. WHERE `COZEMPIC_NUDGE_OFF` is set THEN the nudge segment SHALL be suppressed
while the context/cost segments still render; WHERE `COZEMPIC_STATUSLINE_OFF` is set
THEN the command SHALL print nothing (full opt-out).

R6. The command SHALL be **composable, never clobbering**: `cozempic statusline
--wrap '<user-status-command>'` SHALL run the user's existing status command with
the SAME stdin JSON, capture its stdout, and append (or prepend) Cozempic's segment.

R7. `cozempic init --statusline` SHALL wire the command **opt-in, with consent**:
IF an existing `statusLine.command` is configured THEN init SHALL offer to wrap it
(set `statusLine.command` to `cozempic statusline --wrap '<existing>'`); ELSE it
SHALL set `cozempic statusline`. It SHALL NEVER silently overwrite an existing
status line.

R8. The command SHALL be fast and total: it runs on every status-line refresh
(after each assistant message, after `/compact`, on mode change; debounced 300ms),
so it SHALL avoid heavy work — prefer the supplied `used_percentage`/`cost` over
re-reading the transcript, and SHALL exit 0 with a best-effort line (or empty) on
any error rather than break the user's status line.

R9. (Optional) WHEN the latest turn shows a large `cache_creation_input_tokens`
with `cache_read_input_tokens ≈ 0` on a non-first turn THEN the command MAY render a
small "rebuilt after idle" marker (e.g. `↻`). Low-confidence heuristic — gate behind
a flag; the first turn always writes cache, so do not flag it.

R10. WHERE the user opts into `refreshInterval: N` in settings THEN the command
SHALL remain correct when re-run on a timer during idle (idempotent, no state
mutation required to render).

## Design

- **New command** `cmd_statusline(args)` in `cli.py`, registered in `build_parser`.
  Reads stdin JSON (tolerant of missing/null fields), builds segments, prints one line.
- **Segment builder** (`statusline.py`, new): pure function `render(payload) -> str`
  so it is unit-testable without a live session. Segments: context%, cost, nudge,
  optional cache-miss marker. Color via ANSI; degrade to no-color if `NO_COLOR` set.
- **Percentage source**: prefer `context_window.used_percentage`; fallback to
  `quick_token_estimate(Path(payload["transcript_path"]))` / detected window.
- **Tiers**: reuse `_NUDGE_DEFAULT_TIERS`, `COZEMPIC_NUDGE_PCTS`, and
  `get_session_nudge_tiers` from `cmd_nudge` so the status-line nudge and the
  (legacy) Stop nudge agree. No once-per-tier latch here — the status line is a
  persistent indicator, not a one-shot, so it simply reflects current state.
- **`--wrap`**: read stdin once into a buffer; `subprocess.run(user_cmd, input=buf)`;
  print its stdout, then the Cozempic segment. Quote-safe; on wrapped-command failure,
  still print Cozempic's segment (don't lose the whole status line).
- **`init --statusline`**: read `~/.claude/settings.json` (honoring `CLAUDE_CONFIG_DIR`),
  detect `statusLine`, prompt to wrap-or-set, write atomically (reuse existing atomic
  settings writer). Add to `cozempic doctor` a check that the status line is wired.

## Tasks

- [ ] `statusline.py` `render(payload)` + unit tests (present %, null %→transcript
  fallback, cost present/absent, tier colors, NO_COLOR, opt-outs, empty/garbage stdin).
- [ ] `cmd_statusline` + `--wrap` (+ tests: wrap composes, wrapped failure degrades).
- [ ] Register subparser; reuse nudge-tier helpers.
- [ ] `init --statusline` opt-in wiring (detect existing, wrap-with-consent, atomic
  write, never clobber) + tests.
- [ ] `doctor` check: status line wired? + advise enabling.
- [ ] Docs/README: how to enable, `--wrap`, opt-outs, `refreshInterval` note.

## Out of scope

- Spinner / "thinking" sub-line text — hardcoded in the CC UI, not customizable
  (#50510/#21599/#27982). Confirmed dead end; will not pursue.
- Mid-task visibility (#50679) — not needed; the nudge is an at-prompt indicator.
