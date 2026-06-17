# Cost Analysis — why a free idle/end hook pays for itself

> Pricing below is **illustrative** (representative Opus-class rates,
> `Pin = $5/M`, `Pout = $25/M`). Confirm against live rates before quoting
> numbers. Multipliers and behaviours are sourced from the Anthropic / Claude
> Code docs (see citations at the bottom).

## The asymmetry that drives the whole feature

**The hook is free. The compaction event is not.**

- A `SessionEnd` / idle hook is fire-and-forget: per the
  [context-window docs](https://code.claude.com/docs/en/context-window), plain
  stdout on exit 0 is written to the debug log only — it is **not** added to the
  model context. So cost-logging + compression at end-of-interaction costs
  **0 model tokens**. Its only cost is a sub-second JSONL rewrite.
- A **compaction event** generates a summary ≈12% of the prefix at **output**
  rates, plus a prefix read and a re-cache of the new summary prefix — and then
  imposes downstream rework because the model lost detail.

Because the optimization is free, it never has to "earn back" its own cost.
Every compaction event it defers or prevents is immediate net savings, and the
break-even is simply *"is the session ever resumed / re-triggered?"*

## Unit of savings: one avoided compaction event

For an 800K-token context at the compaction point (illustrative):

| Component | Calculation | Cost |
| --- | --- | --- |
| Read prefix (warm cache, 0.1×) | 800K × 0.10 × $5/M | $0.40 |
| **Generate summary (output, ~12% of prefix)** | 96K × $25/M | **$2.40** |
| Re-cache the new summary prefix (1.25×) | 96K × 1.25 × $5/M | $0.60 |
| **≈ C_compact per event** | | **~$3.40** |
| + downstream rework from fidelity loss | model re-reads files, re-derives | unquantified, often largest |
| + wall-clock stall | "most of compaction's time goes to generating the summary" | seconds per event |

So the relevant quantity is **C_compact ≈ $3.40 + rework + latency**, not the
sub-dollar marginal cache-read shavings a fixed `gentle` trim would yield.

## Expected savings

```
savings ≈ P(prune keeps the re-trigger under the compaction threshold)
          × C_compact
          × (number of re-triggers)
```

Worked example — a PR-webhook workflow whose transcript is replayed and tips
into auto-compact on each of 20 re-triggers:

| | Per re-trigger | × 20 re-triggers |
| --- | --- | --- |
| Unpruned (compacts each time) | C_compact ≈ $3.40 | **~$68** + 20 stalls + 20 lossy summaries |
| Pruned under target (no compaction) | $0 | **$0** |

The marginal cache-read saving I tabled earlier (~$0.17/re-trigger for a 5%
gentle trim) is real but an order of magnitude smaller — it is **not** the case
for the feature. The case is avoiding the discrete compaction events.

## Why "prune to a target," not "trim a fixed %"

A 5% gentle trim on a transcript already at ~90% of the window still gets
compacted on the next re-trigger — you paid the rewrite and saved nothing.
The compressor must get the transcript a **safe margin below the compaction
threshold** (default: ≤55% of the window, the guard's hard1 tier), escalating
`gentle → standard → aggressive` only as far as needed. This is exactly what
Cozempic's guard already does mid-session; the feature reuses it at end-of-
interaction. (See `requirements.md` Req 2.1, `design.md` Component B.)

## Why the trigger is timed to cache expiry

Pruning changes the prefix, so doing it *while a warm cache exists* would forfeit
cheap 0.1× reads on the next turn. The fix is timing, not avoidance:

- Fire the prune only once the conversation has been **idle longer than the
  prompt-cache TTL** (default 300s ≈ the 5-min cache window;
  `COZEMPIC_CACHE_TTL_SECONDS=3600` for the 1-hr cache option).
- At that point the warm cache has **already expired** — there is no hit left to
  forfeit — so pruning is downside-free, and it pre-positions any later
  re-trigger to replay a slim prefix.
- While the session is still active (a message arrived within the TTL),
  compression does **not** fire, so an in-flight cache is never invalidated.

This is why the **idle-past-TTL daemon trigger is primary** and `SessionEnd` is
only a clean-exit fast path — the doc research also found `SessionEnd` fires
unreliably on `/exit` and `/clear`, so coverage cannot depend on it.

## Cost-logging half

Pure upside: reads per-message `costUSD` (or token usage when absent) and appends
one line to `~/.claude/cozempic-metrics/session-costs.jsonl`. 0 model tokens, no
re-trigger dependency, no downside.

## When it does NOT help

- A session that is genuinely one-and-done and never resumed: compression saves
  no tokens (nothing replays it). The only "waste" is a sub-second rewrite + one
  `.bak`. Cost-logging is still free upside.
- Two re-triggers *within* the cache TTL: the idle trigger intentionally won't
  have fired yet, so the warm cache is preserved — correct behaviour.

## Sources

- Hook output not added to context (0 tokens):
  https://code.claude.com/docs/en/context-window
- Compaction summary ≈12% of prefix, reads existing cache, summary-generation is
  the dominant time: https://code.claude.com/docs/en/context-window
- Cache multipliers (write 1.25× / 2.0×, read 0.1×) and TTLs (5 min / 1 hr):
  https://platform.claude.com/docs/en/build-with-claude/prompt-caching
- Resume replays full history; cost scales with conversation length; no cache
  hits behind a changed prefix: https://code.claude.com/docs/en/sessions ·
  https://code.claude.com/docs/en/prompt-caching
- `SessionEnd` reliability gaps on `/exit` / `/clear`:
  https://github.com/anthropics/claude-code/issues/17885 ·
  https://github.com/anthropics/claude-code/issues/6428
