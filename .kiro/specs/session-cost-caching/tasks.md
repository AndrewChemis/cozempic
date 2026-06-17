# Tasks — Session-End Cost Accounting, Cache-Friendly Compression & Supply-Chain Hardening

Implementation plan. Each task lists the requirement IDs it satisfies. Check off
as completed. Tasks are ordered so each builds on the previous and the tree
stays green between steps.

## 1. Cost accounting core

- [ ] 1.1 Create `src/cozempic/cost.py` with `METRICS_DIR`/`COST_FILE`
  constants, a `CostResult` dataclass, and `compute_cost(messages)` that sums
  per-message `costUSD` and falls back to `extract_usage_tokens` +
  `detect_model` when absent. _(Req 1.2, 1.3)_
- [ ] 1.2 Add `record_session_cost(messages, payload)`: build the JSONL record
  (schema in design.md), honour `COZEMPIC_COST_LOG_OFF` / `COZEMPIC_NO_TELEMETRY`
  opt-out, create `~/.claude/cozempic-metrics/` if absent, and append atomically.
  _(Req 1.4, 1.5, 1.8, 1.9)_
- [ ] 1.3 Add per-`session_id` idempotency (tail-scan existing file, skip if
  already recorded). _(Req 1.6)_
- [ ] 1.4 Unit tests: `costUSD` sum, usage-only fallback (`cost_usd=null`),
  duplicate-session no-op, opt-out env, unwritable-dir degrade, atomic-write
  crash-safety. _(Req 1.2–1.9)_

## 2. Cache-aware compression (prune-to-target + idle trigger)

- [ ] 2.1 Create `src/cozempic/compress.py` with `compress_to_target(path,
  messages, snapshot)`: escalate `gentle → standard → aggressive` via
  `run_prescription` only until `estimate_session_tokens` lands ≤
  `target_pct × detect_context_window` (default 55%, env
  `COZEMPIC_SESSION_END_COMPRESS_TARGET_PCT`), with `gentle` as the floor; write
  back through `_PruneLock` + `save_messages(create_backup=True, snapshot=…)`.
  _(Req 2.1, 2.6, 2.8)_
- [ ] 2.2 Handle `PruneLockError`/`PruneConflictError` as clean skips; add the
  already-at-target skip, benefit-floor skip
  (`COZEMPIC_SESSION_END_COMPRESS_MIN_BYTES`), and the
  `COZEMPIC_SESSION_END_COMPRESS_OFF` opt-out. _(Req 2.7, 2.9)_
- [ ] 2.3 Add the **idle-past-cache-TTL** trigger to the guard daemon
  (`src/cozempic/guard.py`): track last-activity timestamp; when `now -
  last_activity > COZEMPIC_CACHE_TTL_SECONDS` (default 300s) AND above target AND
  not `detect_in_flight`, call `compress_to_target`. Fire at most once per idle
  episode (re-arm on new activity). _(Req 2.2, 2.4, 2.5)_
- [ ] 2.4 Unit tests: ladder escalates only as far as needed (gentle suffices →
  stops at gentle; huge session → reaches aggressive); already-at-target skip;
  lock/append-conflict skips; below-floor skip; backup created. _(Req 2.1, 2.6–2.9)_
- [ ] 2.5 Guard-daemon tests: idle past TTL fires once; new activity within TTL
  re-arms and does NOT fire (warm cache preserved); in-flight work defers.
  _(Req 2.2, 2.4)_

## 3. `session-end` command + hook wiring

- [ ] 3.1 Add `cmd_session_end(args)` in `src/cozempic/cli.py`: read stdin
  payload, take `snapshot_session(path)`, `load_messages` once, call
  `record_session_cost` THEN `compress_to_target` (cost first — Req 2.10), always
  exit 0. _(Req 1.1, 1.7, 2.3, 2.10)_
- [ ] 3.2 Register the `session-end` subparser in `build_parser` and wire
  dispatch. _(Req 1.1)_
- [ ] 3.3 Add a `SessionEnd` hook block to `src/cozempic/data/hooks.json` and
  the `plugin/hooks/hooks.json` mirror; bump the embedded `cozempic-hook-schema`
  marker `v13 → v14` everywhere it appears. _(Req 1.1, 2.1)_
- [ ] 3.4 Extend `tests/test_hooks_sync.py` to cover the new block and the
  schema-marker bump (both files identical). _(Req 1.1)_
- [ ] 3.5 Verify `init`/`doctor` re-sync installed hooks on the schema bump
  (smoke check `cmd_init` / `cmd_doctor`). _(Req 1.1)_
- [ ] 3.6 Integration test: feed a fixture payload to `session-end` and assert a
  cost record is written AND the transcript is compressed, with cost reflecting
  pre-strip `costUSD`. _(Req 1.4, 2.8)_

## 4. GitHub Actions SHA pinning

- [ ] 4.1 Resolve the current release tag → 40-char SHA for `actions/checkout`,
  `actions/setup-python`, `actions/upload-artifact`,
  `actions/download-artifact`, `pypa/gh-action-pypi-publish`. _(Req 3.1, 3.2)_
- [ ] 4.2 Pin each `uses:` in the publish workflow to its SHA with a `# vX.Y.Z`
  comment. _(Req 3.1, 3.2)_
- [ ] 4.3 Resolve whether the workflow must move to `.github/workflows/publish.yml`
  to be active; move it (or document why it stays) — OPEN item from design.
  _(Req 3.3)_
- [ ] 4.4 Add a Dependabot `github-actions` config (or a documented refresh
  procedure) so pins update deliberately. _(Req 3.4)_
- [ ] 4.5 Add a guard test/lint asserting every `uses:` is a 40-hex SHA. _(Req 3.1)_

## 5. Hashed Python build tooling

- [ ] 5.1 Generate `packaging/ci/build-requirements.txt` with `--generate-hashes`
  pins for `build`, `setuptools`, `wheel`. _(Req 4.1)_
- [ ] 5.2 Switch the CI build step to
  `pip install --require-hashes -r packaging/ci/build-requirements.txt`. _(Req 4.1)_
- [ ] 5.3 Align `pyproject.toml` `[build-system] requires` to the pinned
  versions; add a note that runtime deps are empty so hashing is build-only.
  _(Req 4.2, 4.3)_
- [ ] 5.4 Document the hash-refresh procedure. _(Req 4.4)_

## 6. Auto-update opt-in flip

- [ ] 6.1 In `src/cozempic/updater.py:maybe_auto_update`, gate the upgrade on
  `COZEMPIC_AUTO_UPDATE`; when unset, emit the throttled manual-upgrade notice
  and return. Keep `COZEMPIC_NO_AUTO_UPDATE` / `COZEMPIC_PIN` short-circuits
  ahead of it. _(Req 5.1, 5.2, 5.3, 5.5)_
- [ ] 6.2 Update the SessionStart upgrade clause in `src/cozempic/data/hooks.json`
  and `plugin/hooks/hooks.json` to require `COZEMPIC_AUTO_UPDATE` non-empty (and
  keep the existing disable/pin guards). _(Req 5.1, 5.4)_
- [ ] 6.3 Update `npm/install.js` `decideInstall` so `--upgrade` is added only
  when `COZEMPIC_AUTO_UPDATE` is set; first-install still works without it.
  _(Req 5.1, 5.4)_
- [ ] 6.4 (Optional, Req 5.7) Add a publish-age window to `_do_upgrade` using
  PyPI `upload_time`. _(Req 5.7)_
- [ ] 6.5 Update README + onboarding copy (and Homebrew `caveats`) to document
  the new opt-in default and `COZEMPIC_AUTO_UPDATE=1`. _(Req 5.6)_
- [ ] 6.6 Tests: default no-upgrade + at-most-one notice/24h; opt-in restores
  upgrade; pin/disable still win; npm `decideInstall` unit test for the new gate.
  _(Req 5.1–5.5)_

## 7. Wrap-up

- [ ] 7.1 Run the full test suite; confirm no regression in existing hook tests.
- [ ] 7.2 Update CHANGELOG / version-sync checklist locations (the repo keeps
  packaging mirrors in lockstep — see `git log` release-sync commits).
- [ ] 7.3 Manual smoke: trigger a real `SessionEnd`, confirm a cost line lands in
  `~/.claude/cozempic-metrics/session-costs.jsonl` and the transcript shrank.

---

### Suggested PR slicing

These are independent and can ship as separate PRs to keep review tight:

1. **PR A** — Tasks 1–3 (cost accounting + compression + SessionEnd hook).
2. **PR B** — Tasks 4–5 (CI/action/build SHA + hash pinning).
3. **PR C** — Task 6 (auto-update opt-in flip; the only user-facing default
   change — gets its own review + changelog note).
