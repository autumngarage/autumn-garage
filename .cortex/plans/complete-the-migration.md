---
Status: Active
Owner: cross-repo (vanguard + outrider)
Parent: `.cortex/plans/separation-finish-line.md` (covers what shipped tonight)
Created: 2026-04-26 evening
Goal-hash: (recompute with cortex doctor)
Cites:
  - `.cortex/plans/separation-finish-line.md`
  - `.cortex/plans/vanguard-db-ownership.md`
  - `.cortex/journal/2026-04-26-b21-foundation-railway-acted.md`
---

# Complete the migration

> **The remaining 30% of the vanguard ↔ outrider separation. Three load-bearing couplings still hold (DB writes both directions, SharedInfra runtime context, exit-reasoning carve-out) plus minor residue. Each needs deliberate motion in a specific order; none is a single mechanical edit. After this plan ships, vanguard fresh-clones without an outrider checkout, both repos run independently, and the AST cluster-boundary guard goes strict-zero on both sides.**

## Where we are

After tonight's two-wave all-nighter:

- AST cluster-boundary allow-list at **63 entries** (was 167). Floor is reached for what's safely deletable.
- All 63 entries are categorized into 4 buckets:
  - **~58 entries** — `outrider.platform.db*` (B.2 / DB writes)
  - **1 entry** — `outrider.agents.shared_infra.SharedInfra` (runtime context)
  - **2 entries** — `outrider.api.{signals, server}` (coupled with SharedInfra)
  - **2 entries** — `outrider.api.{reason_deep, ExitReasoningResult}` (exit-reasoning carve-out)
- Data flow prep in flight (#85 — `trade.resolved` outcome event)
- Postgres-B4xF wired, schema applied, vanguard's DB layer vendored + dormant
- Vanguard pyproject still pins `outrider @ git+https://...@main`. The dep ships.

## What "done" means

1. `outrider @ git+...` removed from `vanguard/pyproject.toml`.
2. `OUTRIDER_PAT` secret + sibling-checkout removed from vanguard CI; `GITHUB_PAT` block removed from `vanguard/railway.json`.
3. AST cluster-boundary tests both sides flip to strict-zero (no allow-list, no exceptions).
4. Vanguard fresh-clone (`git clone vanguard && cd vanguard && uv venv && uv pip install -e .`) bootstraps and runs core tests on a machine with no outrider repo present.
5. Outrider's flywheel learning loop continues working — `daily_eval` reports per-agent PnL via `outcomes_inbox` (HTTP-fed) instead of vanguard's `trades` table.
6. Vanguard runs prod with `DATABASE_URL_VANGUARD` only — no `outrider`-side database connection.
7. Both repos' Railway services deploy independently — no cross-repo build-time secrets, no sibling-checkout coupling.

## The remaining couplings — and why each needs motion

### Coupling A — DB writes (vanguard → outrider's Postgres)

Vanguard writes operational state (trades, cycles, risk_state, system_heartbeats, events_outbox) into outrider's shared Postgres via `outrider.platform.db_queries.*`. Foundation built tonight (Postgres-B4xF + schema + vendored layer); cutover requires data migration + dual-write or stop-the-world swap.

### Coupling B — DB reads (outrider → vanguard's tables)

Outrider's `learning/daily_eval.py` reads vanguard's `trades` table for per-agent PnL. Outrider's `lorien/health.py` reads vanguard's `system_heartbeats` for liveness. These are *reverse leaks* — outrider treating vanguard's operational state as readable infrastructure. Architecturally wrong and a hard blocker for the cutover (vanguard cannot stop writing to shared Postgres while outrider still reads from it).

### Coupling C — SharedInfra runtime context

`outrider.agents.shared_infra.SharedInfra` is a 667-line class instantiated by vanguard's `runner.py:3241`. Vanguard's whole trading stack is built on it: `infra.kalshi`, `infra._schwab`, `infra.risk_guard`, `infra.execution_queue`, `infra.fast_dispatcher`, ~30 attribute accesses. SharedInfra also wraps DB sessions, so it couples tightly with Coupling A. Cannot vendor cleanly without addressing the DB layer in the same wave.

### Coupling D — Exit reasoning

`outrider.api.reason_deep` and `ExitReasoningResult` used by `vanguard/garrison/exit_evaluator.py` + `vanguard/garrison/options_exit.py`. Unresolved product question: does exit reasoning belong to vanguard (execution-side decision; vanguard ships its own LLM call via Conductor) or outrider (research-side decision; vanguard subscribes to it as part of the proposal contract)?

## Approach — five stages, partially sequenced

```
Stage 1: data flow prep                       (in flight tonight as #85)
   │
   ▼
Stage 2: decouple outrider's reads            (Coupling B)
   │
   ▼
Stage 3: vendor SharedInfra into vanguard     (Coupling C)
   │  ┊  (couples with Stage 5 — sessions)
Stage 4: resolve exit-reasoning carve-out     (Coupling D, independent — can run anytime)
   │  ┊
   ▼  ▼
Stage 5: DB cutover                           (Coupling A)
   │
   ▼
Stage 6: drop the dep + verify fresh-clone
   │
   ▼
Stage 7: operational cleanup (orphan tables, scripts, etc.)
```

Stage 4 is independent and can be done in parallel with any other stage.
Stage 3 and Stage 5 are coupled — SharedInfra holds DB sessions, so they ship together as a single wave (call it 3+5).

### Stage 1 — data flow prep (✅ in flight)

`trade.resolved` outcome contract + emit. Vanguard starts POSTing TRADE_RESOLVED events with profit per resolved trade. Outrider's `outcomes_inbox` accumulates the data Stage 2 will read from. **Sub-task #85.**

Acceptance: every vanguard trade resolution lands a TRADE_RESOLVED event in outrider's `outcomes_inbox` within seconds.

### Stage 2 — decouple outrider's reads (Coupling B)

**2a. Migrate `daily_eval.py:630` per-agent PnL.**
Read from `outcomes_inbox` filtered to `outcome_type='trade.resolved'`, group by `payload.proposer`, sum `payload.profit_cents`. Same shape as today's query, different source. Requires Stage 1 data accumulation (≥7 days of trades flowing through outcomes_inbox before the migration is statistically meaningful).

**2b. Drop heartbeat reads (option D).**
`daily_eval.py:740` and `lorien/health.py:562` both call `read_freshest_heartbeat([agent_name])` for vanguard liveness. Architectural answer: outrider doesn't monitor its customer. Vanguard self-monitors via Railway healthchecks + its own alerts; if vanguard goes down, vanguard's own observability catches it.

Delete the calls. If outrider's daily eval needs a "vanguard alive" check, replace with a lightweight "did we receive any TRADE_RESOLVED outcomes in the last 24h" check — that uses outcomes_inbox and is the right signal anyway (heartbeat-without-trades is a false-positive of liveness).

**2c. Defer or delete operational scripts.**
`outrider/scripts/{nullify_corrupt_market_price_at_entry, outcome_audit}.py` read `trades`. These are operational/maintenance scripts, not runtime. Two options:
- Delete them. They were one-shot fixes; the corruption they targeted is historical.
- Migrate to HTTP. Vanguard exposes a narrow `GET /admin/trades` endpoint scoped to a service-account API key (vanguard-as-server smell, but tightly contained).

Recommend delete. They've outlived their reason for existing.

**Outcome of Stage 2:** outrider has zero direct reads of vanguard-owned tables.

### Stage 3+5 — SharedInfra vendoring + DB cutover (Coupling C + A coupled)

These ship together because SharedInfra wraps DB sessions; you cannot move the DB without moving SharedInfra and vice versa.

**3+5.1 — Vendor SharedInfra into vanguard.**
Copy `outrider/agents/shared_infra.py` → `vanguard/runner/shared_infra.py` (or wherever fits vanguard's runtime layout). Drop research-cluster-private behavior (agent instantiation that references research collectors, council, calibration). Keep what vanguard's runner actually uses: broker handles, risk guard, execution queue, fast dispatcher.

DB sessions inside the vendored SharedInfra read `DATABASE_URL_VANGUARD` (already wired on Railway) via `vanguard._platform.db.session_scope` (already vendored).

**3+5.2 — Vendor `outrider.api.signals` (Signal, SignalBus, SignalUrgency).**
Move into `vanguard/router/signals.py` or similar. These are runtime classes for fast_dispatcher; not API shapes (despite the module name). Vanguard owns its own dispatcher signals post-vendor.

**3+5.3 — Delete `outrider.api.server.app` import.**
Vanguard currently boots outrider's FastAPI app in-process for some legacy code path. Post-cutover, vanguard runs only its own runner.py — outrider's API runs in the outrider Railway service. Delete the import + the call site.

**3+5.4 — Resolve `agent_registry.instantiate_agent`.**
Last C.1 entry. Two options:
- **Vendor:** copy the agent registry → vanguard, with the agent classes vanguard cares about (likely a smaller set than outrider's full registry).
- **Drop:** if vanguard's only use of `instantiate_agent` is for proposals it's executing, the proposal payload already carries enough metadata (agent_consensus, agent_focus). Vanguard can attribute trades to agents by name without instantiating their classes.

Recommend drop. Instantiating outrider's agent classes in vanguard's process was always a smell — agents are research-cluster runtime, not execution-cluster runtime.

**3+5.5 — Run DB cutover (B.2.2 + B.2.3 from `vanguard-db-ownership.md`).**

Choose one of two strategies:

**Strategy α — dual-write soak (safer, slower):**
1. Add `VANGUARD_OWN_DB_WRITES` (default 0) and `VANGUARD_LEGACY_DB_WRITES` (default 1) env flags.
2. Wrap every vanguard write site so it can hit one or both DBs based on flag values.
3. Set `VANGUARD_OWN_DB_WRITES=1` on Railway. Vanguard now writes to both DBs.
4. 24-48h soak. Daily reconciliation: row counts + checksum of latest 1000 rows per table.
5. `pg_dump` historical vanguard-owned data from shared Postgres → restore into Postgres-B4xF (so vanguard can read its own history).
6. Set `VANGUARD_LEGACY_DB_WRITES=0`. Vanguard writes only to Postgres-B4xF.
7. 24h validation soak.

**Strategy β — stop-the-world cutover (faster, riskier):**
1. Schedule a low-traffic window (~30 min — overnight or weekend morning).
2. Stop vanguard service. Confirm no in-flight writes.
3. `pg_dump` vanguard-owned tables from shared Postgres → restore into Postgres-B4xF.
4. Deploy vanguard with imports flipped to `vanguard._platform.db*` (already vendored, dormant).
5. Restart vanguard. Verify reads + writes hit B4xF only.
6. If anomalies surface in the first hour, roll back deploy + restore previous state (data is safe in either DB after the dump).

Strategy α is the textbook safe path. Strategy β is faster and works when the data volume is small (it is: 5 tables, small row counts) and a brief vanguard outage is acceptable.

**Decision point for the operator (you):** which strategy. Default recommendation: **β** — vanguard's data volume is low, the tables are append/upsert-style not high-OLTP, and a 30-minute scheduled outage is cheaper than 1-2 weeks of dual-write infrastructure code that you delete immediately after.

**Outcome of Stage 3+5:** vanguard's runtime imports zero `outrider.platform.db*` and zero `outrider.agents.shared_infra`. AST allow-list drops to ~2 entries (the exit-reasoning carve-out only).

### Stage 4 — exit-reasoning resolution (independent)

**Decision point for the operator:** vanguard-side or HTTP?

**Option vanguard-side** — `vanguard/garrison/exit_reasoning.py` shells to Conductor (`conductor call --auto --tags reasoning`). Vanguard ships its own LLM call. Eliminates the only remaining `outrider.api.reason_deep` import.

**Option HTTP** — outrider exposes `POST /v1/exit-reasoning` accepting position state + recent market data, returning `ExitReasoningResult`. Vanguard.contract mirrors the request/response shapes. Vanguard switches to HTTP. The `outrider.api` import disappears in favor of HTTP.

Recommend **vanguard-side** because exit decisions are execution timing, not research analysis. Vanguard already speaks Conductor (the autumn-garage quartet). Architectural cleanest.

If vanguard-side: ~1-day workstream.
If HTTP: ~2-day workstream (new endpoint on outrider, contract mirror on vanguard, integration tests).

**Outcome of Stage 4:** zero `outrider.api.*` imports remain in vanguard.

### Stage 6 — drop the dep

Mechanical PR. Cannot ship until Stages 3+5 and 4 are done.

1. `vanguard/pyproject.toml`: delete `"outrider @ git+https://github.com/outriderintel/outrider.git@main"` line.
2. `vanguard/.github/workflows/test.yml`: drop sibling-checkout step + `OUTRIDER_PAT` secret reference. Drop `OUTRIDER_PAT` from repo secrets.
3. `vanguard/railway.json`: drop `GITHUB_PAT` block from buildCommand.
4. `vanguard/tests/aegis/unit/test_no_outrider_imports.py`: empty the allow-list. Convert the soft test (warns) to a strict test (fails on any outrider import).
5. Run `bash scripts/test-trade-critical.sh` — must stay green.
6. **Verification:** on a machine with no `~/Repos/outrider/` checkout: `git clone https://github.com/.../vanguard && cd vanguard && uv venv && uv pip install -e . && bash scripts/test-trade-critical.sh`. Must exit 0.
7. Update `vanguard/CLAUDE.md` and `vanguard/README.md` to remove sibling-checkout instructions and the `OUTRIDER_PAT` setup section.
8. Single PR through codex review. Title: `feat(separation): drop outrider dep — fully decoupled`.

### Stage 7 — operational cleanup

After Stage 6 ships and vanguard runs cleanly for ~7 days:

1. **Drop orphaned tables** in shared Postgres: `trades`, `cycles`, `risk_state`, `system_heartbeats`, `events_outbox`. Confirm no writes for 7+ days first (audit via `pg_stat_user_tables.last_autovacuum_count`). One-line `DROP TABLE` migration.
2. **Audit shared Postgres** for any other vanguard-only data we missed. Drop or migrate.
3. **Outrider's NEXT_STEPS.md** — update to remove "blocked by separation" framing on WS4/5/6/11/12/13. They unblock.
4. **Master plan** (`full-vanguard-outrider-separation.md`): flip Status to `shipped`. Add final "Updated-by" entry citing this plan.
5. **Journal entry:** end-state declaration. Both repos independent. Customer #2 onboarding now becomes a real workstream.

## Decision points — LOCKED 2026-04-26 evening

Operator principle: **"intelligence stays in outrider, trading goes in vanguard, no redundancy. elegance."**

| Decision | Choice | Reason |
|---|---|---|
| Stage 5 strategy | **β stop-the-world cutover** | Dual-write IS redundancy by definition. β cheapest in code + calendar time. |
| Stage 4 exit-reasoning home | **HTTP endpoint on outrider** | LLM reasoning is intelligence — stays in outrider. Vanguard calls the endpoint and decides whether to act on the reasoning. Same shape as the proposal contract. |
| `agent_registry.instantiate_agent` | **drop** | Instantiating outrider agent classes in vanguard's process violates both "no redundancy" and "intelligence stays in outrider." Agent metadata on the proposal is sufficient for trade attribution. |

The earlier autumn-garage-AI recommendation for vanguard-side exit reasoning was wrong by the principle — overridden.

## Time + sequencing estimate

Assuming the operator makes the three decisions and one execution day per workstream:

| Stage | Wall-time | Wait-time | Notes |
|---|---|---|---|
| 1 — data flow prep | done tonight | ~7 days | Wait for outcomes_inbox to accumulate |
| 2 — decouple outrider reads | 1 day | — | After Stage 1 wait |
| 3+5 — SharedInfra + DB cutover | 2-3 days | 1 day soak (β) or 2-3 days soak (α) | Choose strategy |
| 4 — exit reasoning | 1 day | — | Independent; can parallelize |
| 6 — drop the dep | 0.5 day | — | Mechanical |
| 7 — cleanup | 0.5 day | 7-day post-cutover wait | Final |
| **Total** | **5-6 execution days** | **8-10 calendar days** | β path |

If choose α (dual-write): add ~3 days of code + soak. β recommended.

## Risks + mitigations

| Risk | Severity | Mitigation |
|---|---|---|
| TRADE_RESOLVED event flow has bugs (Stage 1) | Medium | Stage 2 only migrates after 7-day soak with reconciliation (count of trade-resolutions vs count of TRADE_RESOLVED events ingested) |
| Stage 3+5 coupled wave is too big to ship cleanly | High | Split: 3+5.1 (vendor SharedInfra, runtime ctx still uses old DB) ships first as a "no behavior change" PR, then 3+5.5 (DB cutover) flips imports. Two PRs instead of one. |
| Strategy β: data lost during pg_dump→restore | Medium | Verify row counts post-restore. Keep shared-Postgres tables intact (read-only) until Stage 7 cleanup. Rollback path = redeploy previous vanguard image. |
| daily_eval migration loses historical data | Low | Outcomes_inbox historical depth = however long Stage 1 runs before Stage 2. If <30 days at Stage 2 time, outrider's daily_eval can dual-source: outcomes_inbox for new, trades for old, until trades data ages out. |
| Exit-reasoning vanguard-side LLM has worse calibration than outrider's | Medium | A/B test: keep outrider HTTP path available behind a flag for the first 2 weeks of vanguard-side; compare exit timing, drift, win-rate. |
| Outrider's agents need vanguard's heartbeat for some other purpose we missed | Low | Stage 2b grep + lint + manual audit before delete. If something does break, add a `GET /admin/customer-liveness` endpoint as a stop-gap. |

## Success criteria

1. `grep -i "outrider" vanguard/pyproject.toml` returns empty.
2. `grep -i "OUTRIDER_PAT" vanguard/.github/` returns empty.
3. `grep -i "GITHUB_PAT" vanguard/railway.json` returns empty.
4. `pytest vanguard/tests/aegis/unit/test_no_outrider_imports.py -s` reports allow-listed count = 0.
5. Fresh-clone bootstrap on a clean machine: passes.
6. Vanguard's Railway service runs prod with `DATABASE_URL_VANGUARD` only (no `DATABASE_URL` reference to shared Postgres).
7. Outrider's `learning/daily_eval.py` runs successfully reading from `outcomes_inbox` and reports per-agent PnL within 5% of historical figures (sanity check on the migration).
8. Outrider's `lorien/health.py` no longer references vanguard's heartbeats.
9. Both repos' AST cluster-boundary tests are strict-zero on both sides.
10. No regression in vanguard's 600+ trade-critical test count throughout each stage.

## What this plan does NOT cover

- **Customer #2 onboarding (WS13).** Once vanguard runs as a clean HTTP customer, copying the integration shape for a hypothetical second customer becomes a legit workstream. Not part of the migration; a follow-on.
- **Outrider commercialization.** Per-customer auth, rate limiting, billing, status page (WS12, WS6, WS5). All park-listed; revisit after this plan's Stage 7.
- **SharedInfra design pass.** This plan ships a vendored copy with research-cluster-private bits stripped. A future workstream may rewrite SharedInfra from scratch as a vanguard-native runtime context (smaller, simpler, no inherited research-cluster smells). Not gating; ship vendored first, refactor later.
- **Legal Tier 1 publisher posture.** Real concern, orthogonal. Pursue when commercializing.
