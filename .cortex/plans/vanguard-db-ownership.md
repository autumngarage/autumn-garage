---
Status: Active (B.2.1 foundation in flight; cutover stages B.2.2/B.2.3 user-action)
Owner: vanguard
Parent: `.cortex/plans/separation-finish-line.md` SF-4
Created: 2026-04-26
Updated: 2026-04-26 evening (Postgres-B4xF provisioned + DATABASE_URL_VANGUARD wired; B.2.1 foundation PR shipping tonight)
Workstream-id: B.2
---

# Vanguard owns its execution-side database

> The last code-level coupling between vanguard and outrider: vanguard's runtime today reads and writes outrider's Postgres directly through `outrider.platform.db_queries`. This plan migrates those write paths into a vanguard-owned DB so the `outrider` Python dep can finally drop. After this lands, vanguard and outrider share zero process-state.

## Status

**Active.** Foundation work in flight tonight (B.2.1). Postgres-B4xF (PG18) was already provisioned in the `daring-strength` Railway project. `DATABASE_URL_VANGUARD` env var is set on the vanguard service, resolving to Postgres-B4xF via reference variable. Currently no code reads it — the var is dormant until the foundation PR (B.2.1) ships and the cutover PRs (B.2.2 / B.2.3) flip the migration flags. Cutover stages remain user-action because they change live runtime behavior.

## Why

Per the parent separation plan: "vanguard is a clean HTTP customer of outrider; zero in-process Python coupling either direction." Today's reality is that vanguard imports `outrider._platform.db.session_scope` and `outrider._platform.db_queries.{insert_outbox_event, upsert_trade, start_cycle, complete_cycle, write_heartbeat, load_risk_state, save_risk_state, get_trade_by_order_id, get_trade_by_ticker, count_open_options_brackets, ...}` and writes vanguard-owned operational data into outrider's tables. That's the actual dep coupling — every other import has either been migrated to HTTP or deleted.

## What vanguard's DB needs to own

These tables and their write paths are vanguard-owned conceptually (operational state of vanguard's execution loop), even though they currently live in outrider's Postgres:

| Table | Purpose | Current location | Migration target |
|---|---|---|---|
| `outbox_events` | Vanguard → outrider trade outcome relay | outrider DB | vanguard DB |
| `trades` | Vanguard's executed trades + lifecycle | outrider DB | vanguard DB |
| `cycles` | Vanguard run-loop heartbeat / cycle stats | outrider DB | vanguard DB |
| `risk_state` | Vanguard's risk-engine persistent state | outrider DB | vanguard DB |
| `options_brackets` | Vanguard's open options bracket positions | outrider DB | vanguard DB |
| `heartbeats` | Vanguard's liveness signal | outrider DB | vanguard DB |

The following stay in outrider's DB and are reachable only via HTTP from vanguard:

| Table | Why outrider-owned |
|---|---|
| `proposals` (research output) | Outrider's product. Vanguard reads via `GET /v1/proposals`. |
| `agent_predictions`, `agent_calibration`, `agent_track_records` | Research-internal; vanguard reads aggregated views via `GET /v1/agents/*` and `GET /v1/calibration/*`. |
| `clusters`, `cluster_registry` | Research-internal. |
| `markets_kalshi`, `markets_polymarket` | Outrider's roster; vanguard reads via candidate_instruments on the proposal. |
| `flywheel_signals` | Research-internal; populated via outrider's own ingest of vanguard's outbox events. |

## Approach — three sub-stages

Each is a single PR. Sub-stages run sequentially because they touch the same DB sessions.

### B.2.1 — Foundation (in flight tonight)

Prerequisites already satisfied:
- ✅ Postgres-B4xF (PG18) provisioned in `daring-strength` Railway project
- ✅ `DATABASE_URL_VANGUARD` env var set on vanguard service (reference variable to Postgres-B4xF)
- ✅ Postgres-B4xF confirmed empty

Code work (single PR, dormant — no runtime behavior change):
1. Vendor `outrider/platform/db.py` → `vanguard/_platform/db.py`. `session_scope()` reads `DATABASE_URL_VANGUARD` instead of `DATABASE_URL`. Keep SQLAlchemy table definitions only for vanguard-owned tables: `trades`, `cycles`, `risk_state`, `system_heartbeats`, `events_outbox`. Drop every research-internal table.
2. Vendor `outrider/platform/db_queries.py` → `vanguard/_platform/db_queries.py`, scoped to the helpers vanguard actually imports (~18 functions).
3. Apply schema to Postgres-B4xF via psql. Save the DDL as `vanguard/_platform/migrations/0001_initial_schema.sql` for reproducibility.
4. **DO NOT** change any runtime imports. Vanguard's runner.py + garrison/ + transport/ + vault/ keep importing from `outrider.platform.db_queries` UNCHANGED. The new vanguard module exists and works but is dormant until B.2.2.
5. Acceptance: existing 627-test suite stays green; new smoke tests verify the vendored module imports + connects.

The foundation is dormant code. No runtime change. AST cluster-boundary allow-list stays at 63.

### B.2.2 — Activate dual-write + cutover (USER ACTION, daylight)

Sub-stage A — dual-write soak:
1. New PR adds two flags: `VANGUARD_OWN_DB_WRITES` (default 0) and `VANGUARD_LEGACY_DB_WRITES` (default 1). Wraps every vanguard write site so it can hit one or both DBs.
2. Switch vanguard's runtime imports from `outrider.platform.db_queries` → `vanguard._platform.db_queries_dual` (the wrapper). At default flag values, behavior is identical to today.
3. Set `VANGUARD_OWN_DB_WRITES=1` on Railway (keep `VANGUARD_LEGACY_DB_WRITES=1`). Vanguard now writes to BOTH DBs on every operation.
4. 24-48h soak. Add a daily reconciliation script that compares row counts + checksum of the latest 1000 rows per table across both DBs.

Sub-stage B — cutover:
1. After clean soak, set `VANGUARD_LEGACY_DB_WRITES=0`. Vanguard writes only to its own DB.
2. 24h validation soak.
3. If anomalies surface, flip back ON. If clean, proceed to B.2.3.

### B.2.3 — Drop the imports + the dep

After clean cutover soak:
1. Delete every `outrider._platform.db*` and `outrider._platform.db_queries*` import from vanguard. The AST allow-list B.2 row goes empty.
2. Drop the `outrider @ git+...` line from `vanguard/pyproject.toml`. (This is SF-5.)
3. Drop `OUTRIDER_PAT` secret from `vanguard/.github/workflows/test.yml`. Drop the sibling-checkout step.
4. Drop the `GITHUB_PAT` block from `vanguard/railway.json` buildCommand.
5. Verify fresh-clone bootstrap: `git clone https://github.com/.../vanguard && cd vanguard && uv venv && uv pip install -e . && bash scripts/test-trade-critical.sh` exits 0 on a clean machine.
6. Flip `vanguard/tests/aegis/unit/test_no_outrider_imports.py` to a strict no-imports guard (mirror outrider's side).
7. Drop the orphaned tables (`outbox_events`, `trades`, `cycles`, `risk_state`, `options_brackets`, `heartbeats`) from outrider's DB after 7 days of confirmed no-writes.

## Success criteria

1. Vanguard runs end-to-end in prod with `DATABASE_URL_VANGUARD` only — no `outrider`-side DATABASE_URL set on its service.
2. `outrider @ git+...` removed from `pyproject.toml`.
3. AST cluster-boundary test on vanguard side has zero allow-list entries.
4. Fresh-clone bootstrap works.
5. The orphaned tables in outrider's DB stay empty for ≥7 days post-cutover.

## Risks + mitigations

| Risk | Mitigation |
|---|---|
| Cutover loses an in-flight outbox event | Dual-write soak (B.2.1) + reconciliation script catches divergence before cutover. |
| Schema drift between outrider's old definitions and vanguard's vendored copy | Pin Alembic revision at cutover. Vanguard owns its schema going forward; outrider's old definitions are deletable post B.2.3. |
| Outrider has a reader (e.g., dashboards, ad-hoc queries) on vanguard-owned tables we don't know about | Audit before B.2.2. `grep -rn "outbox_events\|trades\|cycles\|risk_state" outrider/` and similar. Anything found that's still alive becomes either an HTTP endpoint vanguard exposes, or is documented as a `research-feedback-path` to be removed in B.2.2. |
| Local dev gets harder (two DBs to set up) | Vanguard's tests already use respx for HTTP and SQLite for in-process DB. Local prod-mirror still needs Postgres, but it's vanguard-only — simpler, not harder. |

## Out of scope

- Any new feature work on either repo. This is a pure migration.
- Multi-tenant DB partitioning. Vanguard owns one DB; if a second customer ships, that customer owns its own DB.
- Read replicas / sharding. Single instance is fine until proven otherwise.

## Operator runbook (for daylight execution)

1. Provision Postgres on Railway under vanguard's project. Get the connection string.
2. Set `DATABASE_URL_VANGUARD` on vanguard's Railway service.
3. Locally, set the same in `.env.local`.
4. Branch `feat/b-2-1-dual-write-db`. Run B.2.1 steps. Test against local Postgres first, then push, then watch Railway logs for clean dual-write.
5. After 24-48h soak, branch `feat/b-2-2-cutover`. Flip flag. Watch.
6. After 24h post-cutover, branch `feat/b-2-3-drop-dep`. Drop imports, dep, secrets. Verify fresh-clone.
7. Merge each PR through codex review per touchstone flow.

Total expected wall-time: ~3-4 working days of execution + 2-3 days of soak windows.
