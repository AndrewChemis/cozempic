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

Biggest payoff for least work: **#1 (doctor TTL check)** + **#2 (post-idle miss
detector)** for scenario B; **#3 (advisor)** for scenario A; **#0** to make the
nudge actually visible; then **#6–#8** cleanups. Note that #0 gates how visible
#2/#3 are — if Stop `systemMessage` stays broken, route the important signals
through `doctor`/`current`, which the user invokes directly and always render.
