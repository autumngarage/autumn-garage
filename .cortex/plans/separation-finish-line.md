---
Status: Active
Owner: autumn-garage (cross-repo coordination)
Parent: `.cortex/plans/full-vanguard-outrider-separation.md`
Created: 2026-04-26
---

# Separation finish line

> The minimum remaining work to declare vanguard and outrider fully decoupled. Once done, both teams work independently and only meet at outrider's HTTP API. Anything not on this critical path is post-separation work and lives on the parking lot.

## Definition of done

1. `outrider @ git+https://github.com/outriderintel/outrider.git@main` removed from `vanguard/pyproject.toml`.
2. `vanguard/tests/aegis/unit/test_no_outrider_imports.py` has **zero** allow-list entries (becomes a strict guard).
3. `outrider/tests/aegis/unit/test_no_vanguard_imports.py` keeps strict (already done).
4. `OUTRIDER_PAT` secret removed from vanguard CI workflows + Railway env.
5. Vanguard fresh-clone bootstrap works end-to-end with no sibling outrider checkout.
6. Each repo ships a `NEXT_STEPS.md` documenting the independent-work roadmap.

## Current state (as of 2026-04-26 evening)

- Vanguard main: `outrider @ git+...` still in `pyproject.toml` (line 13).
- Vanguard main: ~167 outrider import sites (counted via grep, `vanguard/runner.py` alone has ~40).
- AST cluster-boundary guard categorizes every remaining import under exactly one transition workstream: **B.2** (DB), **C.1** (reference data), **C.2** (HTTP API shapes), or **A.1** (research drivers).
- All four channels of `candidate_instruments` populate (kalshi live, polymarket via WS1, options via WS2).
- Agents-roster, strategy-id, conviction-calibration, event-shape, agent-focus, thesis-long, fee-schedule, calibration-categories all live on the wire.

## Remaining work — partitioned by what gates it

### Tonight (code-only, no Railway state changes) — SHIP IN PARALLEL

These are pure code refactors. Vanguard's runtime behavior is unchanged because the imports being removed are either dead code paths, or have already-shipped HTTP/contract-mirror replacements that we just need to swap in.

**SF-1 — finalize C.1 reference-data cutover** (~15 import sites)
- Vendor `outrider._platform.trade_math` → `vanguard/_platform/trade_math.py`. Pure-functional helpers; safe to copy.
- Delete `outrider._platform.trading_profile` call sites — vanguard reads its own profile from env per parent plan §C.1.
- Tighten allow-list `ALLOW_PLATFORM_REF_DATA` accordingly.

**SF-2 — finalize C.2 HTTP API shape cutover** (~80 import sites)
- `from outrider.api import build_event` → `vanguard.contract.events.build_event` (mirror exists post-PR #16).
- `from outrider.api import get_snapshot, estimate_position_probability` → vanguard-local broker reads OR HTTP via existing wrappers.
- `from outrider.api.outcomes` / `outrider.api.outcomes_ingest` → `vanguard.contract.outcomes` (mirror exists post-PR #24).
- `from outrider.api.notifications` → vanguard-local notification helper (currently lazy-imported in `runner.py`; trivially in-lineable).
- `from outrider.api.server` → delete the entry-point shim; vanguard never serves outrider's FastAPI in prod.
- Carve-out: `outrider.api.reason_deep` and `ExitReasoningResult` for `garrison/exit_evaluator.py` + `garrison/options_exit.py` — open product question on whether exit reasoning belongs in vanguard or stays a research call. Keep allow-listed; resolve in daylight hours.

**SF-3 — finalize A.1 research-driver deletions** (~30 import sites)
- Delete: `outrider.autolab.*`, `outrider.quant.backtest_framework`, `outrider.council.market_helpers`, `outrider.forge.signal_bus`, `outrider.agents.shared_infra`, `outrider.flywheel.attribution`, `outrider.flywheel.agent_calibration`, `outrider.strategies.vol_score`.
- All call sites are either dead (the autolab driver was moved into outrider in A.1.1) or replaced by proposal payload reads.
- Update allow-list `ALLOW_RESEARCH_DRIVERS` to empty.

### Daylight (Railway-coordinated) — REQUIRES USER ACTION

**SF-4 — B.2 vanguard owns its execution-side DB**
- Provision Railway Postgres for vanguard.
- Vendor `outrider._platform.db` connection layer → `vanguard/_platform/db.py` (writes against vanguard's DB only).
- Vendor the trade/outbox/cycle/heartbeat/risk-state write paths from `outrider._platform.db_queries` into `vanguard/_platform/db_queries.py`.
- Dual-write transition: vanguard writes to both DBs, then cutover, then drops outrider DB writes.
- Detailed plan: `.cortex/plans/vanguard-db-ownership.md` (written alongside this plan).

**SF-5 — P2.7 drop the dep** (gated entirely on SF-1/2/3 + SF-4)
- Edit `vanguard/pyproject.toml`: delete the outrider line.
- Edit `vanguard/.github/workflows/test.yml`: drop sibling-checkout step + `OUTRIDER_PAT`.
- Edit `vanguard/railway.json`: drop `GITHUB_PAT` block from buildCommand.
- Edit `vanguard/tests/aegis/unit/test_no_outrider_imports.py`: empty the allow-list, flip the guard to strict.
- Verify: fresh `git clone vanguard && cd vanguard && uv venv && uv pip install -e . && bash scripts/test-trade-critical.sh` exits 0 on a machine with no outrider checkout.

### Documentation (tonight)

**SF-6 — `vanguard/NEXT_STEPS.md`** — what vanguard's team works on once the dep drops. Captures execution flywheel (B.2 → SF-4), exit-reasoning carve-out resolution, broker integrations roadmap, paper-to-live promotion criteria, and the tier-1-publisher operational stance.

**SF-7 — `outrider/NEXT_STEPS.md`** — what outrider's team works on once the dep drops. Captures the deferred excellence workstreams (WS4 streaming, WS5 DX, WS6 observability/SLA, WS11 invariant tests, WS12/13 customer onboarding), legal Tier 1 posture work, and the per-customer-auth roadmap.

## Out of scope (parked)

These remain valuable but do **not** finalize separation. Defer until SF-5 ships.

| Item | Why deferred |
|---|---|
| WS4 streaming API | Outrider polish; the proposal-cursor HTTP path is fine for vanguard. |
| WS5 developer experience | Post-separation polish for second-customer onboarding. |
| WS6 observability + SLA | Outrider production hardening; not separation-critical. |
| WS11 Layer 2 invariant tests | The AST guard already catches the load-bearing case. |
| WS12 per-customer auth | No second customer yet; YAGNI until SF-5 lands. |
| WS13 customer onboarding flow | Same. |
| Legal Tier 1 posture | Real concern; orthogonal to code separation. |
| Cortex install in vanguard/outrider | Tooling-side; doesn't move separation forward. |
| Naming cleanup batches 2-5 | In-flight as agent work, but not gating separation. |

## Sequencing

```
SF-1 ┐
SF-2 ┼─→ allow-list shrinks to B.2-only ─→ SF-4 ─→ SF-5 ─→ DONE
SF-3 ┘                                        (Railway)   (dep drop)

SF-6, SF-7 (docs) ship in parallel with SF-1/2/3.
```

Tonight's outcome: AST allow-list contains only B.2 entries. SF-4 becomes the last code surface to wire up. SF-5 is mechanical once SF-4 lands.

## Success criteria

1. `bash scripts/test-trade-critical.sh` stays green throughout each PR.
2. Codex pre-push review passes on every PR (no `--no-verify`).
3. After SF-1/2/3 merge, `pytest vanguard/tests/aegis/unit/test_no_outrider_imports.py -s` shows allow-list count dropped from ~167 to ≤25 (B.2 only).
4. After SF-5 ships: `grep -i outrider vanguard/pyproject.toml` returns empty; `grep OUTRIDER_PAT vanguard/.github/` returns empty.
5. Both `NEXT_STEPS.md` files exist and are honest about what's still load-bearing in each repo.
