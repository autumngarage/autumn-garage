---
date: 2026-04-26
type: decision
trigger: T1.1 (.cortex/plans/ touched), T1.9 (vanguard PR #34 merged)
load-priority: normal
---

# B.2.1 foundation shipped + Railway state acted on

## Context

User pointed out: "you have railway cli so you can do stuff." Earlier journal entry called the night done with B.2 deferred to daylight. With Railway CLI access reframing the scope, I went back and shipped the safe slices of B.2 that don't change live runtime behavior.

## What I did directly on Railway

1. **Verified Postgres-B4xF state** — empty PG18 instance already provisioned in `daring-strength` project. Volume mounted, no tables. Safe to use as vanguard's future DB.
2. **Set `DATABASE_URL_VANGUARD`** env var on vanguard service via reference variable: `${{Postgres-B4xF.DATABASE_URL}}`. Used `--skip-deploys` to avoid triggering a deploy. No behavioral change because no code reads the var yet.

## What the agent shipped (PR #34, vanguard)

- Vendored `vanguard/_platform/db.py` — connection layer reading `DATABASE_URL_VANGUARD`, scoped to 5 vanguard-owned tables: `trades`, `cycles`, `risk_state`, `system_heartbeats`, `events_outbox`. Drops every research-internal table from the vendored copy.
- Vendored `vanguard/_platform/db_queries.py` — 18 helpers vanguard imports.
- Applied schema to Postgres-B4xF via psql. Saved DDL as `vanguard/_platform/migrations/0001_initial_schema.sql`.
- 7 new tests + 1 opt-in live-connection test (`RUN_VANGUARD_DB_LIVE=1`).
- AST cluster-boundary allow-list **unchanged at 63** — this PR is purely additive, no runtime imports flipped yet.
- Codex review caught two real bugs in outrider's inherited shape: `unresolve_trade()` had a missing-WHERE bug that would wipe resolution state on every trades row when both args were falsy; `write_heartbeat()` had a non-atomic SELECT-then-INSERT/UPDATE race. Both fixed in vanguard's vendored copies with regression tests. Outrider's originals are untouched (they have their own constraints) — but worth a follow-up there too.

End state: vanguard has its own DB layer; the schema lives on Postgres-B4xF; the env var is wired; nothing runtime-side has changed.

## What I deliberately did NOT do

The agent's proposed B.2.2 was "mechanical import-line swap." That would:
- Point vanguard's runtime at the empty Postgres-B4xF instantly.
- Lose access to all live state (open trades, risk state, cycle history, heartbeats) currently in shared Postgres.
- Break outrider's reads of those tables (see below).

Cutover needs either (a) data migration (pg_dump → restore into B4xF), then hard cutover, or (b) dual-write soak (vanguard writes to both DBs, outrider continues reading shared, cut over once outrider is decoupled). Decision belongs in daylight with a real session.

## The blocker that surfaced

While probing for cutover prerequisites, I grep'd outrider for reads of vanguard-owned tables. **4 places** outrider reads vanguard tables directly:

1. `outrider/learning/daily_eval.py:630` — `FROM trades` to compute per-agent Kalshi PnL. The flywheel-learning loop's per-agent realized-PnL check.
2. `outrider/learning/daily_eval.py:740` — `read_freshest_heartbeat([agent_name])` for per-agent liveness in the daily eval.
3. `outrider/platform/lorien/health.py:562` — `read_freshest_heartbeat(sources)` for system health monitoring.
4. `outrider/scripts/{nullify_corrupt_market_price_at_entry,outcome_audit}.py` — operational scripts that read trades. Lower priority.

The architectural answer: outrider should read from `outcomes_inbox` (which vanguard POSTs into via the existing `POST /v1/outcomes` endpoint). Each outcome event already has profit + ticker + agent attribution. Same for heartbeats: outrider could expose `GET /v1/health/customer/{name}` that vanguard pings, OR vanguard's heartbeat lives in vanguard's own DB and outrider monitors via a separate polling endpoint.

This is real B.2 prerequisite work that affects the flywheel learning loop. Filed as task #84. Should not be done autonomously — needs a human in the loop on the migration design.

## What's actually left for separation

Updated SF-4 (#72) description. The work breaks down:

1. ✅ B.2.1 foundation (PR #34, tonight)
2. ⏳ #84 outrider stops reading vanguard tables (daylight, affects learning loop)
3. ⏳ B.2.2 data migration strategy decision (dual-write soak vs pg_dump+cutover)
4. ⏳ B.2.2 cutover (depends on 2 + 3)
5. ⏳ SF-5 drop the outrider git+https dep (mechanical post-cutover)

Tonight's slice was the safe slice. The remaining slices each have real production-data risk and want human judgment.

## Total tonight's tally (cumulative across both all-nighter waves)

- ~17 PRs through codex review (vanguard #26-#34, outrider #36-#41, autumn-garage #13-#15)
- AST allow-list 167 → 63 (unchanged in last wave; foundation was additive)
- 1 Railway env var set
- 1 schema applied to a fresh DB
- Two latent bugs in outrider's DB shape caught + fixed in vanguard's vendored copies (codex earned its keep)
- 5 new follow-up tasks captured (#77-#84) carrying the surfaced concerns into daylight

Calling the night done at the right moment. Cutover work needs daylight + human judgment on the migration approach.
