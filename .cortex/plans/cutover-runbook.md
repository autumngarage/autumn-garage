---
Status: Active (executable when prerequisites land)
Owner: autumn-garage (operational, my hands)
Parent: `.cortex/plans/complete-the-migration.md` Phase 3
Created: 2026-04-26
---

# Cutover runbook — vanguard DB swap

> **Stop-the-world cutover from shared Postgres → Postgres-B4xF. ~5-15 min wall-time. Vanguard is briefly stopped during the data dump+restore. Mechanical exits in outrider don't fire during this window — vanguard isn't reading proposals, isn't placing orders, isn't relevant.**

## Prerequisites (all must be done before run)

- [ ] `outrider @ git+...@main` still pinned in vanguard (this is the LAST run with it pinned)
- [ ] All non-DB outrider imports already migrated: sigma deleted, signals vendored, notifications-recon deleted, ingest_outcome HTTP, exit-reasoning HTTP
- [ ] `outrider.api.{get_snapshot, estimate_position_probability}` audit resolved (HTTP or delete)
- [ ] `outcomes_inbox` backfilled with synthesized `TRADE_RESOLVED` events from historical `trades` (so `daily_eval.py:630` migration has data)
- [ ] `daily_eval.py:630` migrated to read from `outcomes_inbox`
- [ ] AST cluster-boundary allow-list contains only `outrider.platform.db*` entries (B.2 row only)
- [ ] PR with the import-flip ready in branch (vanguard runtime imports flip from `outrider.platform.db*` → `vanguard._platform.db*`)
- [ ] Postgres-B4xF schema applied (✅ done in B.2.1 PR #34)

## Connection strings

> **Never paste live credentials here.** Pull connection strings from Railway at run-time:
>
> ```bash
> SHARED_URL=$(railway variables --service Postgres --kv | grep '^DATABASE_PUBLIC_URL=' | cut -d= -f2-)
> B4XF_URL=$(railway variables --service Postgres-B4xF --kv | grep '^DATABASE_PUBLIC_URL=' | cut -d= -f2-)
> ```
>
> - Shared Postgres (source): `Postgres.DATABASE_PUBLIC_URL` → `ballast.proxy.rlwy.net:51204/railway`
> - Postgres-B4xF (target): `Postgres-B4xF.DATABASE_PUBLIC_URL` → `yamabiko.proxy.rlwy.net:25354/railway`

## Tables to migrate

Per data-ownership doc:

| Table | Strategy |
|---|---|
| `trades` | full dump + restore |
| `cycles` | last 30 days |
| `risk_state` | latest row (PK-ordered) |
| `system_heartbeats` | last 7 days |
| `events_outbox` | drain first, then full |

## Sequence

### Step 1 — drain outbox

Outbox events that are still pending (not yet delivered to outrider via HTTP) need to land before cutover.

```bash
# Connect to shared (source) and check pending outbox
psql "$SHARED_URL" \
  -c "SELECT COUNT(*) FROM events_outbox WHERE delivered_at IS NULL;"
```

If non-zero: trigger vanguard's outbox-relay loop to flush. Check Railway logs to confirm drain completes.

### Step 2 — pre-deploy verify Postgres-B4xF schema

```bash
psql "$B4XF_URL" -c "\dt"
# Expect 5 tables: cycles, events_outbox, risk_state, system_heartbeats, trades
```

### Step 3 — pause vanguard

Via Railway CLI:

```bash
cd ~/Repos/vanguard
railway service vanguard
railway down  # or set a maintenance flag the runner respects
```

Confirm via Railway dashboard or `railway status` that the service is stopped.

### Step 4 — pg_dump + restore

```bash
# Dump vanguard-owned tables from shared
pg_dump "$SHARED_URL" \
  --table=trades \
  --table=cycles \
  --table=risk_state \
  --table=system_heartbeats \
  --table=events_outbox \
  --data-only \
  --column-inserts \
  > /tmp/vanguard-data-$(date +%Y%m%d-%H%M%S).sql

# Restore into Postgres-B4xF
psql "$B4XF_URL" < /tmp/vanguard-data-<TIMESTAMP>.sql
```

For `cycles` + `system_heartbeats` (last-N-days windows): use `--where` filter on the `pg_dump`. Or dump full + delete old rows post-restore. Decide at run-time based on row counts.

### Step 5 — verify restore

```bash
# Counts should match dump
psql "$B4XF_URL" \
  -c "SELECT 'trades' AS t, COUNT(*) FROM trades
      UNION ALL SELECT 'cycles', COUNT(*) FROM cycles
      UNION ALL SELECT 'risk_state', COUNT(*) FROM risk_state
      UNION ALL SELECT 'heartbeats', COUNT(*) FROM system_heartbeats
      UNION ALL SELECT 'outbox', COUNT(*) FROM events_outbox
      ORDER BY 1;"
```

Compare against shared:

```bash
psql "$SHARED_URL" \
  -c "SELECT 'trades' AS t, COUNT(*) FROM trades
      UNION ALL SELECT 'cycles', COUNT(*) FROM cycles
      UNION ALL SELECT 'risk_state', COUNT(*) FROM risk_state
      UNION ALL SELECT 'heartbeats', COUNT(*) FROM system_heartbeats
      UNION ALL SELECT 'outbox', COUNT(*) FROM events_outbox
      ORDER BY 1;"
```

Counts MUST match (or differ only by the time-windowed filters).

### Step 6 — merge the import-flip PR

Vanguard's import-flip PR (already up, awaiting cutover) merges. New code:
- imports from `vanguard._platform.db` and `vanguard._platform.db_queries`
- reads `DATABASE_URL_VANGUARD` (already set on Railway)
- AST allow-list flipped to strict (zero entries)

### Step 7 — restart vanguard

```bash
cd ~/Repos/vanguard
railway service vanguard
railway up  # or release the maintenance flag
```

### Step 8 — validate (first 30 minutes)

- [ ] Railway logs show vanguard connected to Postgres-B4xF (look for "DATABASE_URL_VANGUARD" in startup)
- [ ] First trade cycle completes successfully
- [ ] First outbox event delivered to outrider (check `outcomes_inbox` count incremented)
- [ ] No DB connection errors in logs
- [ ] No FK violations or constraint errors
- [ ] First heartbeat written to Postgres-B4xF.system_heartbeats

If anything looks off → ROLLBACK (Step R).

## Step R — rollback (if needed in first 30 min)

1. `railway down` vanguard
2. Revert the import-flip PR (`gh pr revert <PR>` or merge a revert commit)
3. `railway up` vanguard — runs against shared Postgres again with old code

Data in Postgres-B4xF is left intact (won't get out-of-sync because vanguard isn't writing to it). Can retry cutover after fixing whatever broke.

## Step 9 — observe (first 24 hours)

- Vanguard runs normally, all writes go to Postgres-B4xF
- Shared Postgres' vanguard tables: no new writes (old data remains, treated as cold)
- Outrider's daily_eval reads from `outcomes_inbox` (already migrated)
- No cross-cluster reads/writes anywhere

## Step 10 — drop the dep (PR #73)

After 24h clean operation:
- Edit `vanguard/pyproject.toml` — delete `outrider @ git+...` line
- Edit `vanguard/.github/workflows/test.yml` — drop sibling-checkout + `OUTRIDER_PAT`
- Edit `vanguard/railway.json` — drop `GITHUB_PAT` block from buildCommand
- AST allow-list `test_no_outrider_imports.py` — empty, strict guard
- Verify fresh-clone bootstrap on a clean machine
- Update `vanguard/CLAUDE.md` + `README.md` — drop sibling-checkout references
- PR through codex

## Step 11 — cleanup (post 7-day soak)

- Drop orphaned tables in shared Postgres: `DROP TABLE trades, cycles, risk_state, system_heartbeats, events_outbox;`
- Remove `OUTRIDER_PAT` repo secret from vanguard's GitHub Actions
- Remove `GITHUB_PAT` from Railway buildCommand env
- Update `outrider/NEXT_STEPS.md` to reflect post-separation state
- Master plan `full-vanguard-outrider-separation.md` → Status: shipped
- End-state journal entry

## Risk register

| Risk | Likelihood | Mitigation |
|---|---|---|
| Data loss during dump-restore | Low | Source data left intact; can retry |
| Schema mismatch (vanguard expects column outrider doesn't have) | Low | B.2.1 vendored from outrider's exact definitions; PR #34 verified import compatibility |
| FK violations during restore | Medium | Use `--data-only` to avoid schema commands; tables don't reference each other in vanguard's slice |
| Vanguard fails to start with new DB | Low | DATABASE_URL_VANGUARD already validated in B.2.1 smoke tests |
| Outrider writes to one of vanguard's tables we missed | Low | Audited; we only found read-side leaks (now fixed). If a write surfaces post-cutover, it'll error and we investigate. |
| Outbox drain doesn't complete in step 1 | Medium | Explicit halt + count check; can drain post-cutover too if needed (events_outbox in B4xF) |

## Time estimate

- Step 1 drain: 1-5 min
- Step 2 verify: 30 sec
- Step 3 pause: 30 sec
- Step 4 dump+restore: 1-3 min (small data)
- Step 5 verify: 30 sec
- Step 6 PR merge: 1-2 min
- Step 7 restart: 30 sec
- Step 8 validate: ongoing for 30 min

**Total stop-the-world window: ~5-15 minutes.**
