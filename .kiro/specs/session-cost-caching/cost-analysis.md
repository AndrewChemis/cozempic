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

## Doing the compaction ourselves (resume backstop)

The same asymmetry powers the resume backstop. Native compaction is an LLM
summarization (output-rate tokens, lossy); a Cozempic prune is a mechanical file
op (0 LLM, structure-preserving). On resume of a bloated session we therefore
prune to target ourselves *before* native auto-compaction would fire:

| | Native auto-compaction | Cozempic prune + reload |
| --- | --- | --- |
| What runs | LLM summary ≈12% of the **large** prefix | mechanical strip/dedup, no LLM |
| Token cost | output-rate summary + prefix read + re-cache | cache-creation over the **small** pruned prefix |
| Fidelity | lossy paraphrase | structure preserved (drops redundancy) |

Either way the mechanical prune is cheaper than the LLM compaction. The open
question (which `cost.compress_to_target` branch is the common case) is only
*how* we apply it on resume — in place if the hook runs before ingestion, else
`guard --reload-self` — not *whether* it is cheaper.

## Common misconception: compaction is a cache MISS, not a HIT

A locally-compacted transcript does **not** earn a cache hit. A hit requires the
prefix to **exactly match** a live server-side cache entry; compaction *changes*
the prefix, which guarantees a **miss** (a cache write). So pruning does not
*avoid* the rebuild charge — it makes the **unavoidable** rebuild **cheaper**
(fewer tokens reprocessed) and lossless (no LLM summary). The only thing that
truly avoids the charge is sending the identical prefix while the cache is still
warm (resume before expiry / the 1-hour TTL) — the opposite of compacting.

Three distinct cost events, often blended into one:

| Event | Fires when | Cost |
| --- | --- | --- |
| Cache rebuild (reprocess prefix) | ANY resume after TTL expiry, at any context % | 1.25×/2.0× input over the prefix |
| Native LLM compaction | only near the context **limit** (~90%+) | output-rate summary ≈12% of prefix |
| Local prune ("pre-compaction") | whenever we choose | free, mechanical, no LLM |

In a 50%-context / idle / resume scenario there is **no native compaction** — the
only cost is the cache rebuild; compaction-avoidance is a bonus reserved for
near-full sessions.

**Billing reframe.** On a Claude **subscription**, usage is plan-inclusive (no
per-token charges) and the TTL is 1 hour, so the dollar value here is ~nil — the
win is **latency** and not overflowing into billed usage credits (which also drop
you to the 5-min TTL). Real **dollar** savings concentrate on **API-key /
Bedrock / Vertex** billing (per-token, 5-min default).

## Cost-logging (split out)

The session-end cost-to-file idea is now a separate feature/PR
(`claude/session-cost-logging`). It's pure upside (0 model tokens, no re-trigger
dependency) but unrelated to the cache economics analyzed here.

## When it does NOT help

- A session that is genuinely one-and-done and never resumed: compression saves
  no tokens (nothing replays it). The only "waste" is a sub-second rewrite + one
  `.bak`.
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
