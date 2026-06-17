# Evidence Trail — Session-End Cost & Cache-Rebuild Reduction Investigation

Status: **investigation / feasibility** record. Captures what was researched, what
was confirmed, the assumptions made, the open questions, the points where the
requester challenged the analysis, and the final feasibility verdict. Companion to
`requirements.md` / `design.md` / `tasks.md` / `cost-analysis.md`.

---

## 1. How the goal evolved

1. **Initial ask** — three things: (a) write total session cost to a file outside
   the repo on session end; (b) use Cozempic on session end to compress so future
   re-triggers cache cheaper; (c) supply-chain hardening (SHA-pin actions/packages,
   stop auto-update abuse).
2. **Refined** — fire compression when an interactive session goes idle past the
   prompt-cache TTL, so a later resume replays a slim prefix.
3. **Sharpened (requester)** — the real lever is avoiding the expensive **compaction /
   cache-rebuild**, not marginal cache-read savings; the hook is free, the rebuild is not.
4. **Pinned scenario (requester)** — API-key / usage billing, 5-minute default cache;
   session at ~50% context; user idles >5 min then resumes; wants the rebuild *reduced*
   (not the whole transcript reprocessed), on a **left-open / walk-away** live session.
5. **Verdict** — see §7. The exact target (automatic + local + reduce-on-live-walk-away)
   is blocked by Claude Code's in-memory session architecture.

---

## 2. Research conducted (6 sub-agent passes, official docs + GitHub issues only)

| # | Topic | Key result |
|---|---|---|
| A | SessionEnd hook & cost basics | SessionEnd is fire-and-forget (0 model tokens); fires unreliably on `/exit`,`/clear`; compaction summary ≈12% of prefix |
| B | Resume-hook timing | **Never returned / abandoned** — became moot for the walk-away case (no `--resume` involved) |
| C | Prompt-caching economics | 5-min (1.25× write) vs 1-hour (2.0× write) TTL, both 0.1× read; sliding window; eviction → full reprocess on next turn |
| D | Hook context injection | **No** hook/plugin can inject/replace the transcript or supply a custom compaction summary; only additive `additionalContext` |
| E | Transcript storage & resume | `~/.claude/projects/<slug>/<id>.jsonl`; **resume reads fresh from disk**; external edits unsupported; web = cold VM per trigger |
| F | Subscription 1-hour TTL verify | **Confirmed**: subscription auto-uses 1-hour TTL free; telemetry vars do NOT affect caching; several silent downgrades exist |
| G | Live-session memory vs disk | **DECISIVE**: live session is in-memory; disk edits ignored until `/clear`/`/compact`/exit+resume; exit clobbers external edits |

Plus a **code review** of Cozempic itself (`cmd_reload`, `_spawn_watcher`): reload = prune
+ wait for `claude` to exit + relaunch `claude --resume` in a new terminal; exit is
forced via `/exit` keystroke injection (tmux/screen only) or user action; bails over SSH.

---

## 3. Confirmed facts (with sources)

**Caching economics**
- Cache write **1.25×** (5-min TTL) / **2.0×** (1-hour TTL); cache read **0.1×** both. TTL is a
  **sliding window** reset on each hit; after expiry the next request reprocesses the full
  prefix (cache write). — platform.claude.com/docs/.../prompt-caching; code.claude.com/docs/en/prompt-caching
- **Subscription (Pro/Max/Team/Enterprise)** auth auto-uses the **1-hour TTL, free**.
  `FORCE_PROMPT_CACHING_5M=1` downgrades any auth; `ENABLE_PROMPT_CACHING_1H=1` upgrades
  **API-key/Bedrock/Vertex/Foundry only**. — code.claude.com/docs/en/prompt-caching
- Silent downgrades to 5-min not visible in env: plan-overage → usage credits; **subagents
  always 5-min**; reported server-side regression (GitHub #46829, unverified).
- Telemetry/privacy vars (`DISABLE_TELEMETRY`, `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`,
  `DO_NOT_TRACK`) do **not** affect caching. — code.claude.com/docs/en/env-vars
- Each Claude-Code-on-web trigger = fresh VM, **cold cache**. — code.claude.com/docs/en/claude-code-on-the-web

**Hooks**
- SessionEnd/Stop stdout is **not** added to model context (debug log only); only
  `hookSpecificOutput.additionalContext` costs tokens. → a logging/compression hook is **0 tokens**.
- **No** hook injects/replaces the transcript; `SessionStart.additionalContext` **adds**, never
  replaces; `PreCompact` cannot supply a replacement summary (feature reqs #13170, #27461 closed
  as duplicates, not implemented); no `compactionProvider`/`summaryProvider` plugin API. — code.claude.com/docs/en/hooks
- SessionEnd fires unreliably on `/exit` (#17885) and `/clear` (#6428).

**Sessions / transcript**
- Transcript at `~/.claude/projects/<encoded-cwd>/<session-id>.jsonl`, written continuously.
- **Resume reads the transcript fresh from disk** → an external rewrite IS honored on the
  next resume (a fresh process). Plain resume reuses the session ID; `--fork-session` makes a new one.
- External editing is **unsupported**; minimal validation; must keep valid JSON-per-line,
  no empty text blocks, tool_use/tool_result pairing (#54988).
- **Live (running) session is in-memory authoritative**: a mid-session disk edit is NOT
  reflected on the next turn; takes effect only after `/clear`, `/compact`, or exit+resume;
  **exiting flushes memory to disk and clobbers external edits**. (#51214 in-memory cache
  divergence; #49903 "in-memory state overwriting disk".)

**Cozempic reload (code)**
- `cmd_reload` + `_spawn_watcher`: prune → watcher waits for `claude` PID to exit → opens a
  **new terminal** and runs `claude --resume`. Exit forced via tmux/screen `/exit` injection,
  else user types `/exit`. **Returns early over SSH** (no GUI terminal). It does **not** mutate
  live memory — it performs an **exit+resume cycle**.

---

## 4. Assumptions (not fully verified)

- **Dollar figures are illustrative** (e.g. Opus input $5/M, output $25/M). The *multipliers*
  (1.25/2.0/0.1×) are confirmed; exact per-model $ should be checked against live pricing.
- **Subscription-vs-API-key detection from the hook env** — assumed feasible heuristically;
  the exact reliable signal is **unconfirmed**.
- **Auto-compaction threshold** — "approaches capacity," exact % not published (override:
  `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE`).
- **#46829 1h→5m regression** — reported in a GitHub issue, **unverified by us**; treat as a risk signal.
- **`gentle` ≈ 5%, `standard` ≈ 30%** prune fractions — illustrative; calibrate per session via `cozempic diagnose`.

---

## 5. Open questions

- **(Moot)** Resume-hook-vs-ingestion ordering — agent never returned; only mattered for the
  closed-then-reopened resume backstop, not the walk-away case.
- Reliable detection of the in-force cache TTL (auth mode + overage + subagent + regression).
  Likely needs **empirical measurement** from transcript `usage` (cache_read vs cache_creation).
- Whether a **semi-automatic nudge** (detect idle → prompt user to reload / set 1h TTL) is an
  acceptable UX substitute for silent action.

---

## 6. Where the requester challenged the analysis (and the corrections)

1. **"5-min and 1-hour prices are not the same."** Correct. Fixed: write 1.25× vs 2.0×; reads 0.1× both.
2. **"Avoid the cache hit / send it up for a cache hit."** Corrected the framing: a locally
   compacted transcript **changes the prefix → guaranteed cache MISS**, never a hit. Pruning
   **reduces** the unavoidable rebuild, it does not **avoid** it. (`cost-analysis.md` "Common misconception".)
3. **My claim "prune during idle is sufficient and independent of hook timing."** The requester
   was right to doubt it. For a **live** session this is **false** — memory is authoritative, so a
   disk prune does nothing until a reload. Corrected throughout.
4. **"Confirm subscription = 1-hour, and that telemetry vars don't disable it."** Verified true
   (agent F), with the silent-downgrade caveats surfaced.
5. **"This feature sounds impossible without rewriting live in-memory data, which violates earlier
   fixes / could halt Claude for security."** **Confirmed correct** — see §7. This is the load-bearing
   conclusion and it came from the requester, validated by agent G + the Cozempic code review.

---

## 7. Feasibility verdict

**Target:** fully automatic + local + *reduce the rebuild* on a *left-open / walk-away* live session.
**Verdict: NOT achievable** with current Claude Code hooks/architecture, because:

1. Live session context is **in-memory authoritative**; disk prunes are ignored until
   `/clear` / `/compact` / exit+resume, and exit **clobbers** the edit.
2. **No hook/plugin API** can inject a pre-compacted transcript or rewrite the outgoing request.
3. Applying a pruned transcript to a live session requires an **exit+resume**, which Cozempic
   can only force via `/exit` injection (tmux/screen) or user action, spawning a **new GUI
   terminal** — not automatic, not seamless, not SSH/headless/web-safe.
4. Forcibly mutating in-memory session state is unsupported process manipulation that Claude
   Code's integrity protections resist and could halt the session.

In short: a local tool **cannot reduce a live session's rebuild without an exit+resume cycle**,
and that cycle cannot be driven automatically and non-disruptively from outside the session.

---

## 8. What remains feasible (the salvageable subset)

- **Cost logging at session end** — fire-and-forget, 0 tokens, pure upside. (Split
  out to its own feature/PR: `claude/session-cost-logging`.)
- **Compression for the closed-then-reopened pattern** — `claude --resume` reads the pruned
  file fresh, so a smaller rebuild lands automatically. Works *if the user exits and resumes*
  (not the walk-away pattern).
- **Semi-automatic idle nudge** — detect idle-past-TTL via the guard daemon and **prompt** the
  user to `/cozempic reload` or enable the 1-hour TTL. Cozempic already has the `nudge` channel.
- **Avoid the rebuild via config** — recommend `ENABLE_PROMPT_CACHING_1H=1` (API-key/Bedrock/
  Vertex). On a subscription it is already 1-hour and free. This is the only true *avoid*, and
  it needs no Cozempic.
- **Supply-chain hardening** (Requirements 3–5) — entirely independent of all the above and
  still valid.
