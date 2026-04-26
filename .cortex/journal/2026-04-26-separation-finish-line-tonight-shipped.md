---
date: 2026-04-26
type: decision
trigger: T1.1 (.cortex/plans/ touched), T1.9 (multiple PRs merged across vanguard + outrider)
load-priority: normal
---

# Separation finish-line — tonight's wave shipped

## What happened

Continued tonight's all-nighter into a focused finish-line wave per the human's directive: "immediately prioritize all tasks that finalize the separation, then write next-steps documentation in each repo for their independent work."

Wrote two new plans (`separation-finish-line.md`, `vanguard-db-ownership.md`), restructured the task graph so deferred excellence workstreams explicitly block on SF-5 (dep drop), and shipped the code-only finish-line work in parallel agents. End-of-wave state: every remaining outrider import in vanguard is genuinely gated on B.2 (Railway DB provision) or a documented carve-out — no more low-hanging fruit.

## What shipped

**Outrider** (post-naming + post-orphan-cleanup):
- PR #36 — delete orphan `outrider/quant/backtest_framework.py` (-1080 lines)
- PR #37 — naming batch 2: `outrider/conductor/` → `outrider/scheduler/`, `ARES_*` env → `AUTOLAB_*`
- PR #38 — Shape 3 P1.3: comprehensive contract conformance test for v2.1.0 PublicProposal (32 fields + drift guards)
- PR #39 — naming batch 3: `outrider/_platform/` → `outrider/platform/` + glossary doc
- PR #40 — `outrider/NEXT_STEPS.md` (post-separation roadmap; deferred WS4/5/6/11/12/13 + legal Tier 1 + cortex install)
- PR #41 — fix factual path errors in NEXT_STEPS.md (CONTRACT.md path, test path)

**Vanguard** (finish-line code work):
- PR #26 — Shape 3 P2.2 verifier: insight-field-read regression guard (sentinel values, defensive httpx-patch)
- PR #27 — URGENT mechanical sweep: `outrider._platform.*` → `outrider.platform.*` (45 files, 176/176 symmetric, fixed the import-time break that PR #39 introduced)
- PR #28 — SF-3: A.1 research-driver deletions (`forge.signal_bus`, `council.market_helpers`, `quant.backtest_framework`); ALLOW_RESEARCH_DRIVERS 11→1
- PR #29 — SF-1: C.1 reference-data finalization (vendored `_platform/trade_math.py`; trading_profile already vanguard-local)
- PR #30 — `vanguard/NEXT_STEPS.md` (post-separation roadmap; B.2, exit-reasoning carve-out, broker coverage, Tier 1 publisher operational stance)
- PR #31 — fix factual issues in NEXT_STEPS.md (test count, plan path)
- PR #32 — SF-2 retry: documented every remaining `outrider.api.*` site with its real workstream tag (B.2 / A.1 / carve-out / transition); zero pure-shape swaps remained

**Autumn-garage** (coordination):
- New: `.cortex/plans/separation-finish-line.md` — SF-1..SF-7 punch list with definition-of-done
- New: `.cortex/plans/vanguard-db-ownership.md` — B.2 detail plan, 3 sub-stages, operator runbook for daylight
- Updated: `.cortex/state.md` — hero block reflects finish-line state
- Updated task graph — 8 new tasks (#69-#76, #77-82 follow-ups), parking-lot tasks (#44/48/49/50/60/62/63/64) explicitly blockedBy SF-5

## The headline finding

SF-2's retry surfaced the most important finding of the night: **the AST allow-list at 63 entries is the floor.** Every remaining outrider import in vanguard is one of:

1. **B.2 (DB)** — vanguard's runtime writes operational state into outrider's Postgres. Most of the 63. Cannot drop without owning a vanguard DB.
2. **Exit-reasoning carve-out** — `outrider.api.reason_deep` and `ExitReasoningResult` in `garrison/exit_evaluator.py` + `garrison/options_exit.py`. Open product question.
3. **SharedInfra (A.1)** — `outrider.agents.shared_infra.SharedInfra` is a 667-line runtime context that vanguard/runner.py builds the entire trading stack onto (~30 attributes). Vendoring or rewriting it is its own workstream (#81), likely couples with B.2 since SharedInfra holds DB session refs.
4. **outrider.api.signals** — `Signal`, `SignalBus`, `SignalUrgency` runtime classes used by `router/fast_dispatcher.py`. Coupled with SharedInfra.
5. **outrider.api.server.app** — vanguard's runner currently boots outrider's FastAPI in-process. Collapses when outrider runs out-of-process (post-B.2).

There are no more "easy" deletions or swaps. The path to zero is B.2.

## What's next (daylight, user-action)

1. Provision Postgres on Railway under vanguard's project. Set `DATABASE_URL_VANGUARD`.
2. Walk through `vanguard-db-ownership.md` — B.2.1 dual-write infrastructure (vendor session_scope + write helpers, run dual-write soak), B.2.2 cutover, B.2.3 drop the dep + secrets + sibling-checkout.
3. SF-5 (mechanical PR to flip allow-list to strict + drop `outrider @ git+...` from `pyproject.toml` + drop `OUTRIDER_PAT` from CI/Railway) ships in the same wave as B.2.3.
4. After SF-5: vanguard fresh-clone bootstraps with no outrider checkout. Both repos work independently.

Total wall-time to true separation: ~3-4 days execution + 2-3 days soak per the B.2 plan.

## Followups carried forward

- **#81 SharedInfra workstream** — vendor or rewrite. Coupled with B.2.
- **#82 agent_registry residue** — last C.1 entry. Small cleanup, in flight as I write this.
- **#77 AgentConsensus drift** — vanguard.contract types `agreeing_agents/disagreeing_agents` as `list[str]` but the wire is `list[{name, focus}]`. Small fix, in flight as I write this.
- **#44 Legal Tier 1 posture** — orthogonal to code separation; pursue when commercializing.
- **#48-50, 60, 62-64** — excellence workstreams (streaming API, DX, observability/SLA, Layer 2 invariants, cortex install, customer auth/onboarding) — explicitly blocked on SF-5; revisit after dep drops.

## What I deliberately did NOT do tonight

- Did not start B.2 code without Railway DB provisioning. Speculative migration code without the actual DB target would diverge from reality.
- Did not break the AST allow-list semantics by removing entries that still represent live runtime paths. Conservative bias kept B.2-flagged imports allow-listed; SF-2's retry confirmed this was the right read.
- Did not touch the outrider repo's research-internal machinery (autolab, flywheel, council, council reasoning, agent calibration) — those stay in outrider's pipeline by design.
- Did not write speculative customer-onboarding (WS12/13) before a real second customer surfaces. Defer until needed.
- Did not pre-emptively commercialize the publisher posture — the legal opinion belongs in daylight with a real lawyer.

## Note on the all-nighter format

This is the second consecutive all-nighter on the separation effort (4-25 evening through 4-26 evening, ~24h elapsed wall-time, ~50+ PRs through codex review across the four-tool quartet's production). The output justifies the format: separation went from "many leaks, no plan" to "one plan, one workstream remaining (B.2), daylight runbook in hand." Continuing into the night past this point would mean writing speculative B.2 code without the Railway target — diminishing returns. Calling the wave done at the right moment.
