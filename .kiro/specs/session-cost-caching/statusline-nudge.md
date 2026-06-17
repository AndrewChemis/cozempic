# Feature Spec — `cozempic statusline` (nudge + cost in the Claude Code status line)

The one clearly-feasible, high-value feature to come out of this investigation.
It replaces the unreliable Stop-hook `systemMessage` nudge (§0 of `future-work.md`)
with the Claude Code **status line** — a first-class, reliably-rendered surface —
and shows a live running-cost segment at the same time (distinct from the
session-cost *file* logging, which is its own PR — this is a visible widget).

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

### Integration with `ccstatusline` (the popular status-line tool)

Many users already run **ccstatusline** (sirmalloc/ccstatusline, MIT, wired as
`statusLine.command: npx -y ccstatusline@latest`). Cozempic SHALL integrate with it
rather than clobber it, and fall back to standalone only when it is absent.

**Confirmed mechanism** (github.com/sirmalloc/ccstatusline, docs/USAGE.md):
- ccstatusline has a **`Custom Command`** widget — runs a shell command on every
  refresh, passes the **full Claude Code status JSON on stdin** (plus a
  `terminal_width` field), renders stdout inline, kills on `timeout` (default
  1000ms), preserves ANSI when `preserveColors: true`.
- Config at `~/.config/ccstatusline/settings.json` (honors `CLAUDE_CONFIG_DIR`),
  externally JSON-editable (TUI optional), atomic saves, leaves file untouched if
  invalid. Schema: `{ lines: [ { widgets: [ {type, metadata, foregroundColor,…} ],
  padding, separator } ], …globals }`. No published JSON schema (inferred from TS types).
- ccstatusline already ships **Context %, Session Cost, Cache Read/Write, Cache Hit
  Rate, Compaction Counter** widgets — so Cozempic must contribute ONLY the nudge,
  not duplicate these.

R11. `cozempic init --statusline` SHALL **detect ccstatusline** (e.g. the
`statusLine.command` in `~/.claude/settings.json` references `ccstatusline`, or its
config file is present) and branch:
  - **ccstatusline present** → register a Cozempic **custom-command segment/widget**
    inside ccstatusline's own config (with consent), so Cozempic composes *into* the
    user's existing status line. Cozempic SHALL NOT replace the `statusLine.command`.
  - **ccstatusline absent** → offer the standalone path: set `statusLine.command` to
    `cozempic statusline`, or `--wrap` an existing custom command (R6/R7).

R12. WHEN integrating into ccstatusline THEN Cozempic's segment SHALL render **only
its value-add** — the **reload nudge** (and optional cache-miss marker) — and SHALL
NOT duplicate context%/cost segments that ccstatusline already provides. A
`cozempic statusline --segment nudge` mode SHALL emit just that segment for use as a
custom-command widget. (Standalone mode still renders the full context% + cost + nudge.)

R13. WHERE Cozempic edits ccstatusline's config THEN the write SHALL be atomic,
reversible, and idempotent (re-running init does not add duplicate widgets — detect
an existing Cozempic widget by its command), and SHALL never corrupt or reorder the
user's existing widgets. IF the config is missing/invalid/unrecognized THEN init
SHALL fall back to printing manual instructions rather than editing blindly.

R14. The widget registered SHALL invoke the **installed binary** — `cozempic
statusline --segment nudge` (NOT `npx cozempic`; cozempic is a PyPI/Python package),
with metadata `{ "timeout": 2000, "preserveColors": true, "maxWidth": 40 }` and a
warning-color default. `cozempic init --uninstall-statusline` SHALL remove exactly
that widget.

R15. **Performance** — the segment command runs on EVERY status-line refresh under a
~1–2s timeout, so cold-start matters. `cozempic statusline` SHALL minimize import/
startup cost (lazy imports, no auto-init, no network, prefer the supplied
`used_percentage`/`cost` over re-reading the transcript) and exit fast; if it risks
exceeding the budget it SHALL still print a best-effort/empty line rather than hang.
Detection (R11) SHALL key on `statusLine.command` containing `ccstatusline` and/or the
presence of `~/.config/ccstatusline/settings.json`.

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
- **ccstatusline branch** (R11–R15): detect ccstatusline (by `statusLine.command`
  substring and/or `~/.config/ccstatusline/settings.json`), and when present insert a
  `Custom Command` widget into the primary line's `widgets` array:
  ```json
  { "type": "Custom Command",
    "metadata": { "command": "cozempic statusline --segment nudge",
                  "timeout": 2000, "preserveColors": true, "maxWidth": 40 },
    "foregroundColor": "yellow" }
  ```
  Atomic + idempotent (skip if a widget with that command exists) + reversible
  (`--uninstall-statusline`); printed manual instructions if the config is
  invalid/unrecognized. Three install outcomes total: (a) integrate into ccstatusline,
  (b) standalone `cozempic statusline`, (c) `--wrap` an existing custom command.
- **Cold-start budget** (R15): the segment runs under ccstatusline's ~1–2s timeout on
  every refresh — keep `cozempic statusline` import-light (lazy imports, no auto-init,
  no network) and prefer the stdin `used_percentage`/`cost` over reading the transcript.
- **`--segment <name>`**: render a single named segment (`nudge`, `context`, `cost`,
  `cache`) so Cozempic can be embedded as a widget without duplicating what the host
  status line already shows.

## Tasks

- [ ] `statusline.py` `render(payload)` + unit tests (present %, null %→transcript
  fallback, cost present/absent, tier colors, NO_COLOR, opt-outs, empty/garbage stdin).
- [ ] `cmd_statusline` + `--wrap` (+ tests: wrap composes, wrapped failure degrades).
- [ ] Register subparser; reuse nudge-tier helpers.
- [ ] `init --statusline` opt-in wiring (detect existing, wrap-with-consent, atomic
  write, never clobber) + tests.
- [ ] ccstatusline detection + custom-widget registration (atomic/idempotent/
  reversible; manual-instructions fallback) + `--segment nudge` mode + tests.
- [ ] `doctor` check: status line wired? + advise enabling.
- [ ] Docs/README: how to enable, `--wrap`, opt-outs, `refreshInterval` note.

## Out of scope

- Spinner / "thinking" sub-line text — hardcoded in the CC UI, not customizable
  (#50510/#21599/#27982). Confirmed dead end; will not pursue.
- Mid-task visibility (#50679) — not needed; the nudge is an at-prompt indicator.
