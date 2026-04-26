---
Status: Proposed (activates when user provisions Railway DB for vanguard)
Owner: vanguard
Parent: `.cortex/plans/separation-finish-line.md` SF-4
Created: 2026-04-26
Workstream-id: B.2
---

# Vanguard owns its execution-side database

> The last code-level coupling between vanguard and outrider: vanguard's runtime today reads and writes outrider's Postgres directly through `outrider._platform.db_queries`. This plan migrates those write paths into a vanguard-owned DB so the `outrider` Python dep can finally drop. After this lands, vanguard and outrider share zero process-state.

## Status

**Proposed.** Activates after the user provisions a Postgres instance on Railway under the vanguard service (or sets `DATABASE_URL_VANGUARD` to a managed Postgres elsewhere). Until then, the rest of the separation finish-line plan (SF-1, SF-2, SF-3, SF-6, SF-7) ships independently.

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

### B.2.1 — Provision + dual-write infrastructure

User-action prerequisite: Railway DB instance for vanguard with `DATABASE_URL_VANGUARD` set on vanguard's Railway service (and on local `.env.local` for the local dev path).

Code work:
1. Vendor `outrider/_platform/db.py` → `vanguard/_platform/db.py`. The new module's `session_scope()` reads `DATABASE_URL_VANGUARD` instead of `DATABASE_URL`. Keep the SQLAlchemy table definitions for the vanguard-owned tables; drop the research-internal ones from the vendored copy.
2. Run vanguard's migrations against the new DB to create the table schema. (Vanguard adopts Alembic at this stage if it doesn't already have it; copy the relevant migration files from outrider for the owned tables.)
3. Vendor the relevant write helpers from `outrider/_platform/db_queries.py` → `vanguard/_platform/db_queries.py`. Keep only the helpers vanguard actually calls (audit via grep against runner.py + garrison/*.py). Drop everything research-side.
4. Switch vanguard's call sites to import from `vanguard._platform.db` and `vanguard._platform.db_queries`. Add a runtime feature flag `VANGUARD_DUAL_WRITE_DB=1` (default ON during transition) that writes to BOTH the vanguard DB and outrider's DB on every operation. This guarantees zero data loss while validating the new path.
5. Acceptance: vanguard starts up, both DBs populate identically over a 24h soak. Add a daily reconciliation script that compares row counts + checksum of the latest 1000 rows per table.

### B.2.2 — Cutover to vanguard-only writes

After 1-2 days of clean dual-write soak:
1. Flip `VANGUARD_DUAL_WRITE_DB=0` on Railway. Vanguard now writes only to its own DB.
2. Outrider stops reading vanguard-owned tables. Anywhere in outrider that referenced `outbox_events` etc. is either redirected to vanguard's HTTP endpoint (if it's a research-feedback path) or deleted.
3. Run for 24h. If anomalies surface, flip the flag back ON.

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
