# Vanguard ↔ outrider decoupling foundation shipped — intelligence layer clean, DB layer next

**Date:** 2026-04-26
**Type:** decision
**Trigger:** T1.3 (plan-stage transition: Stages A/B/C-partial/D landed; Stage C.3 deferred)
**Cites:** plans/full-vanguard-outrider-separation, plans/vanguard-execution-flywheel-stage-2-db-ownership, outrider#14-#26, vanguard#11-#23, autumn-garage#7-#11

> An overnight run landed 30+ PRs across outrider, vanguard, and autumn-garage. The decoupling foundation is solid at the intelligence layer — proposal-as-product is real, the contract mirrors are in vanguard, the in-process flywheel is gone, reference data flows over HTTP, and AST cluster-boundary tests guard both directions. The symbolic ship — dropping the `outrider @ git+https` dep from vanguard's pyproject — did not happen tonight. It is gated on B.2 (vanguard owning its own DB schema) and the remaining C.2 test rewrites. Honest accounting below.

## Context

The full vanguard ↔ outrider separation plan (`plans/full-vanguard-outrider-separation.md`) was filed on 2026-04-25 and flipped to `active` later the same day. The plan's "why" was concrete: vanguard imported outrider in 34 non-test source files across 195 import sites and 24 distinct outrider modules. Outrider imported vanguard once (a reverse leak in `outrider/_platform/config_loader.py:647`). The architectural framing — *outrider is the product; vanguard is the canonical customer; the proposal is the entire intelligence payload* — was settled the same day after a major reframing pass.

Tonight's session executed Stages A, B, and most of C in one go. Stage C's symbolic ship (drop the dep) was always going to be the last PR; it remains gated.

## What shipped

**Outrider** (PRs #14–#26):

- #14 — reverse-leak fix in `_platform/config_loader.py` (Stage A.2.1). Outrider stops importing from vanguard.
- #15 — outrider gains its own autolab scheduler (A.1.1). Vanguard's scheduler stops driving research-domain experiments.
- #17, #18, #19 — `GET /v1/reference/trading-profile`, `GET /v1/strategies/{id}`, `GET /v1/reference/fees` (Stage B.1). Reference-data endpoints exist with contract tests.
- #20 — `run_shadow_check_cycle` top-level entry exposed for outrider's scheduler to drive autolab.
- #21, #22, #23 — Shape 3 structural reshape: `PublicProposal` schema bumped to v2.0.0; the audit-of-current-vs-Shape-3 plan; the implementation. `candidate_instruments` (kalshi / polymarket / options channels) lands on the proposal payload. Insight-only fields (`predicted_probability_ci`, `conviction`, `edge_vs_consensus`, `time_horizon`, `agent_consensus`, `event_shape`) wired up.
- #24 — `GET /v1/calibration/categories` (calibration-category lookup; supports the C.2 carve-outs vanguard tests still rely on).
- #25 — outrider AST cluster-boundary test: `outrider/**.py` is forbidden from importing `vanguard.*`. Currently empty allow-list; the reverse-leak fix in #14 made this enforceable.
- #26 — `GET /v1/agents/roster` endpoint. Replaces vanguard's last `outrider._platform.agent_registry` import for runtime.

**Vanguard** (PRs #11–#23):

- #11 — minimum-viable trade-execution platform (`vanguard/_platform/` with `kill_switch.py`, `state_store.py`, `config_loader.py`, `alerts.py`). Stage B.3.
- #9 — vendored 6 stable utilities (`atomic_write`, `paths`, `git_sha`, `exceptions`, `broker_exceptions`, `api_keys`) from `outrider/_platform/`. Stage B.4.
- #12, #15 — `vanguard/contract/` — pydantic v2 mirrors of outrider's HTTP API data types, then bumped to v0.4.0 / `schema_version 2.0.0` to match outrider's Shape 3 reshape. Stage B.5.
- #13 — autolab orchestration deleted from `vanguard/conductor/scheduler.py` (paired with outrider#15). Stage A.1.2.
- #16 — P2.4 audit-and-delete pass on redundant outrider imports. Pure deletes; no behavior change.
- #20 — P2.5 reference-data via HTTP. `vanguard/_platform/fees.py`, `trade_math.py`, `trading_profile.py` switched to HTTP-client wrappers with startup cache. Stage C.1.
- #17, #18, #19 — P2.6 batches: HTTP transport tests respx'd onto `vanguard.contract`; resolution + outcomes tests respx'd (with one bug fix surfaced); deeper-coupling tests gained explicit TODOs against the carve-outs. Stage C.2 partial.
- #21 — execution-flywheel stage 1: in-process flywheel deleted (-1227 LOC). Calibration goes over HTTP. The single largest behavioral subtraction of the night.
- #22 — vanguard AST cluster-boundary test with allow-list. The headline metric below.
- #23 — `vanguard.agents.roster` reads from outrider's `/v1/agents/roster` rather than importing `outrider._platform.agent_registry`.

**Autumn-garage** (PRs #7–#11):

- #7 — flipped the full-separation plan to `active`.
- #8 — pre-commit hygiene (shellcheck SC2120).
- #9 — major architectural reframe: outrider-as-product, Shape 3 proposal, the proposal is the intelligence payload.
- #10, #11 — bar elevated; nine excellence workstreams filed as deferred follow-ups.

## The seven architectural moves

1. **Proposal-as-product.** Outrider's customer-facing API surface collapsed from "lots of endpoints, customer assembles" to "the proposal is everything you need." Internal analysis (vol_score, OQS, Shapley, autolab, calibration) stays inside outrider's pipeline; vanguard stops importing it.
2. **`candidate_instruments` block.** The proposal payload now carries up to 3 expression channels per opportunity (kalshi, polymarket, options). Vanguard picks zero, one, or multiple. Decoupled the *insight* from the *expression*.
3. **`vanguard.contract` mirrors `outrider.api`.** Standard customer pattern: hand-mirrored pydantic v2 models pinned to outrider's `schema_version`. A future contract-conformance test against a fresh outrider response will catch drift.
4. **In-process flywheel decoupled.** Vanguard's calibration/forensic/flywheel machinery (-1227 LOC in PR #21 alone) is gone. Calibration metrics flow over HTTP. Vanguard's own learning loop (per-broker fill quality, per-strategy realization, paper/live drift) is now a separately-scoped follow-up.
5. **Reference data via HTTP.** Fees, trade-math constants, trading-profile, agent roster — all over HTTP with startup-cache. Outrider can rotate any of these without forcing a vanguard redeploy.
6. **AST guards both directions.** `vanguard/tests/aegis/unit/test_no_outrider_imports.py` (with shrinking allow-list) + `outrider/tests/aegis/unit/test_no_vanguard_imports.py` (currently empty). Regressions surface as failed CI on the introducing PR.
7. **Naming sharpened.** The "intelligence vs trading" cluster split is finally legible. Outrider does research and emits proposals; vanguard takes proposals and trades. No machinery shared in either direction at the cluster boundary.

## Honest accounting — what did NOT ship

**The symbolic ship — dropping the `outrider @ git+https` dep from `vanguard/pyproject.toml` — did not happen tonight.** It is gated on:

- **B.2 — vanguard owning its own DB schema.** ~60 import sites in vanguard still hit `outrider._platform.db` and `outrider._platform.db_queries`. These are the events_outbox / outcomes_inbox / proposal cursors / risk state SQLAlchemy models. Vanguard's Postgres database is already separate (`vanguard-db`); the *models* still live in outrider's package. Migrating them is a multi-stage workstream (schema migration prep, dual-write deploy, cutover reads), filed tonight as `plans/vanguard-execution-flywheel-stage-2-db-ownership.md`.
- **Remaining C.2 test rewrites.** ~23 `outrider.api.*` imports in tests, plus a handful of carve-outs (calibration, reasoning) that need either HTTP-mock rewrites or a sharper architectural decision about whether the test belongs in vanguard or outrider.
- **Operational consequences while the dep sticks around.** `vanguard/railway.json` keeps the `GITHUB_PAT` plumbing in its buildCommand. `vanguard/.github/workflows/test.yml` keeps the sibling-checkout step + `OUTRIDER_PAT` reference. None of this can come out until the dep is dropped.

## The headline metric

`vanguard/tests/aegis/unit/test_no_outrider_imports.py` carries an explicit allow-list of permitted `outrider.*` import paths. The allow-list is the trend-toward-zero metric: every cleared category shrinks it. Tonight's pass dropped the count from the original ~195+ legacy import sites to **130 allow-listed entries** at session end. Categories breakdown:

- ~42 `outrider._platform.db_queries` (Stage B.2)
- ~18 `outrider._platform.db` (Stage B.2)
- ~23 `outrider.api.*` (Stage C.2 — tests)
- ~5 `outrider.flywheel.agent_calibration` (carve-outs in tests)
- ~30 small categories (autolab residuals, signal_bus, council helpers, etc.)
- ~12 in C.2-eligible test files awaiting respx rewrite

When the allow-list reaches zero, the dep can be dropped, the railway/CI plumbing comes out, and Stage D.2 (final verification + status flip) runs. The shrinking count is the canonical "are we there yet" signal — checked into git, visible per-PR.

## What's next

- **Stage B.2 (vanguard-execution-flywheel-stage-2-db-ownership)** — filed tonight as `proposed`. Outlines schema migration prep, dual-write deploy, cutover reads, drop the dep. Multi-deploy workstream.
- **Stage C.2 finish** — the test-side carve-outs (calibration, reasoning) plus the remaining `outrider.api.*` test imports.
- **Stage C.3 (the symbolic ship)** — last PR of the migration. Drops the dep, removes the `GITHUB_PAT` block from `railway.json`, removes the `OUTRIDER_PAT` step from CI. Becomes possible the moment the allow-list is empty.
- **Excellence workstreams** — 9 follow-up plans (polymarket coverage, options structuring, agents endpoint v2, streaming API, developer experience, observability + SLA, `thesis_long` generation, `strategy_id` population, conviction calibration curve) ship on top of the now-clean foundation. Each is independently shippable.

## Consequences / action items

- [x] Filed `plans/vanguard-execution-flywheel-stage-2-db-ownership.md` (Status: proposed) as the next workstream toward Stage C.3.
- [ ] Stage B.2 lands → allow-list drops by ~60 → dep can be dropped (Stage C.3).
- [ ] When Stage C.3 lands, write the closing journal entry (`journal/2026-XX-XX-vanguard-outrider-fully-separated.md`) per Stage D.2; flip parent plan to `shipped`.
- [ ] Update `.cortex/state.md` next session with the post-overnight quartet status (this entry can be cited from there).
