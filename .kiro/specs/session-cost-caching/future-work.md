# Future Work — features & fixes for interactive (pre-reload / idle) sessions

Queued backlog from the investigation. Companion to `evidence-trail.md`. None of
this requires the (blocked) live-session auto-reduce; everything here respects the
constraint that a local tool can only use **reload (exit+resume)**, **nudge
(passive FYI)**, and **doctor/telemetry (advice)** channels.

---

## 0. CONFIRMED BUG — the nudge may never reach the user

**Symptom:** "I'm not sure the nudge hook is working."

**Finding:** The `cozempic nudge` CLI is healthy — tested with a synthetic 60%
transcript it emits the correct `{"systemMessage": "…"}` and latches once-per-tier.
The problem is downstream: **Claude Code does not reliably render a top-level
`systemMessage` from a Stop hook** in recent versions.

- `systemMessage` is the documented, user-only field (correct choice in theory) —
  code.claude.com/docs/en/hooks
- Reported regressions (verify against your CC version):
  - **#50542** — Stop-hook `systemMessage` from plugin `hooks.json` silently
    ignored (v2.1.114; UI-side regression).
  - **#40380** — bare `systemMessage` dropped unless wrapped in `hookSpecificOutput`.
  - **#16289** — SubagentStop `systemMessage` not displayed.
  - **#9610** — Stop `systemMessage`s accumulate instead of replacing.

**The bind:** the suggested workaround (wrap in `hookSpecificOutput.additionalContext`)
makes the message **model-visible** — it costs tokens and the model may *act* on
"you should reload," which is wrong for a passive user nudge.

**Options to fix (pick after testing on the target CC version):**
1. Detect CC version; use top-level `systemMessage` where it renders, fall back
   to a wrapped form only where it doesn't.
2. Emit `systemMessage` AND a minimal `hookSpecificOutput` envelope (per #40380)
   without a model-actionable instruction.
3. Move the nudge to a channel that does render (e.g. surface on the next
   `SessionStart`/`PostCompact` additionalContext as an FYI), accepting the delay.
4. Accept that the nudge is best-effort and lean on `doctor`/`current` (which the
   user runs explicitly and always renders) for the important advice.

---

## RECOMMENDED PRIMARY SURFACE — the status line (replaces the broken nudge)

The Stop-hook `systemMessage` channel is unreliable (§0). The **status line** is a
first-class, reliably-rendered surface and is the better home for the nudge **and**
a live cost display at once (the cost *display* is distinct from the session-cost
*file* logging, which is its own PR).

**Why it fits (corrected from the research agent's over-cautious verdict):** the
agent flagged it "problematic" only because of #50679 — the status line is hidden
*during long task execution*. That is irrelevant here: we surface the nudge at the
`❯` prompt, which is exactly when the status line IS shown and exactly the moment a
user decides to reload or is returning from idle. For both target scenarios
(pre-reload ~50%, returning-from-idle) it is reliable where `systemMessage` is broken.

**The payload does the work for us.** The status-line stdin JSON already includes
(code.claude.com/docs/en/statusline):
- `context_window.used_percentage` (+ `current_usage` cache_read/cache_creation split)
- `cost.total_cost_usd`, `total_duration_ms`
- `transcript_path`, `session_id`, `model`, `exceeds_200k_tokens`, `rate_limits`

So a new **`cozempic statusline`** command:
- reads stdin, prints a compact segment e.g. `↯ 62% · $0.34`, and above a tier
  appends a colorized nudge `⚠ /cozempic reload` (yellow ≥55%, red ≥80%);
- surfaces the **post-idle cache-miss** signal directly from `current_usage`
  (high `cache_creation`, ~0 `cache_read` ⇒ "rebuilt after idle");
- falls back to reading `transcript_path` only when `used_percentage` is null
  (pre-first-API-call). ANSI color, emoji, multi-line, OSC-8 links all supported.

**Constraints / design:**
- Exactly ONE `statusLine` command; **no plugin composition API**. So Cozempic must
  be **opt-in and composable**: a `--wrap '<user existing status cmd>'` mode that runs
  the user's command, captures its output, and appends Cozempic's segment.
  `cozempic init --statusline` detects an existing `statusLine` and offers to wrap it
  **with consent** — never silently clobbers.
- Optional `refreshInterval: N` in settings re-runs it during idle (for a live clock /
  "cache likely expired" countdown), at the cost of a periodic exec.
- This consolidates **cost display + context % + reload nudge + cache-miss FYI** into
  one zero-token, never-model-visible surface. Promote above the Stop-hook nudge.

**Spinner — NOT a viable channel.** `spinnerVerbs: {mode, verbs}` customizes only the
**top-line verb word**; the whimsical sub-line messages (the actual "Discombobulating…"
venue Kickbacks.ai uses) are **hardcoded in the React UI and not customizable**
(#50510, #21599, #27982 — won't-do/duplicate). The verb also shows *during work*, not
at the prompt — wrong moment for a reload nudge. Dead end for a CLI/plugin.

## (B) Idle / 5-minute cache-miss sessions — highest value, most feasible

1. **`cozempic doctor` cache-TTL check** *(do first).* Detect auth mode + whether
   the 1-hour TTL is in force; if on API-key/5-min, advise `ENABLE_PROMPT_CACHING_1H=1`.
   Warn about silent downgrades (`FORCE_PROMPT_CACHING_5M`, plan-overage→credits,
   subagents always 5-min, the #46829 regression — check via `/usage`). Pure advice,
   no live mutation; the actual fix for "cache miss after 5 minutes."
2. **Post-idle cache-miss detector → timed nudge.** On the Stop hook after a gap,
   read the last turn's `usage`: if `cache_creation_input_tokens` spiked and
   `cache_read ≈ 0`, a rebuild was just paid. Surface "that turn rebuilt N tokens
   after idle — enable 1h TTL or reload." Data is already in the transcript.
   (Renders only if §0 is solved — otherwise show it via `doctor`/`current`.)

## (A) Pre-reload ~50% interactive sessions

3. **Cost/benefit reload advisor.** Replace "you're at 60%" with the tradeoff from
   `cost-analysis.md`: "reloading re-caches a ~X-token leaner prefix (~$Y) and
   avoids the native compaction you'll hit in ~Z turns." Informed decision, not a guess.
4. **Cache-aware reload timing.** The guard's auto-reload is itself an exit+resume
   that re-caches — time it to fire just *after* the cache would expire on idle so
   it doesn't discard a still-warm cache.
5. **`cozempic current` cache visibility.** Show last-turn cache split (read vs
   creation) + estimated `$`, so users can see when they pay rebuilds. Today
   `current` shows tokens but not the cache/cost breakdown.

## Bug / robustness cleanups

6. **Stop-hook stdin drain.** In `data/hooks.json` (+ plugin mirror) the Stop hook
   does `cozempic nudge || python3 -m cozempic nudge`. If the first binary exists,
   consumes stdin, then errors, the fallback gets **empty stdin** and no-ops.
   Capture `$HOOK_INPUT` to a temp/var and feed both attempts.
7. **Auto-init on read-only commands.** Running `cozempic current` (observed) wires
   7 global hooks into `~/.claude/settings.json` via auto-init — surprising for a
   read-only verb. Gate auto-init off `current`/`diagnose`/`list`, or announce loudly.
8. **Nudge heuristic fallback.** When the transcript tail has no `usage` block,
   `cmd_nudge` returns silently. Fall back to the heuristic token % so fresh
   sessions still nudge.

---

## Suggested order

Biggest payoff for least work:
1. **`cozempic statusline`** (the recommended primary surface) — it simultaneously
   fixes the broken nudge (§0), delivers a live cost display, and carries the
   context-% + cache-miss signals, all from the status-line payload. Highest leverage.
2. **#1 (doctor TTL check)** + **#2 (post-idle miss detector)** for scenario B.
3. **#3 (advisor)** for scenario A — its output now has a reliable home (the status line).
4. **#6–#8** cleanups.

The status line supersedes the §0 dilemma: instead of fighting the unreliable Stop
`systemMessage`, route the nudge + cost + cache-miss signals through the status line
(reliable at the prompt) and keep `doctor`/`current` for explicit on-demand detail.
